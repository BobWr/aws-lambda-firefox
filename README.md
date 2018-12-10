# aws-lambda-firefox
use xvfb firefox ffmpeg in aws lambda

前一段时间，做了一个对浏览器录制推流的镜像，由于对资源的要求比较高，所以需要服务动态伸缩。后发现aws的一款免费服务lambda，号称可以最高一千个实例，并且免费，就尝试迁移到其上面。研究了几天，最终效果差强人意，现做个记录。

1.问题
lambda想要提供的应该只是简单的服务部署，比如一个spring boot项目，它本意可能并不想用户用其来部署耗费大量cpu或者内存资源的应用，其容器内部做了很多限制，比如不可以直接yum install。由于这些限制，之前调试通过的镜像则不可以简单的复制到lambda中，其中最大的问题就是xvfb，firefox，ffmpeg不可通过install的方式下载安装，但可以将其编译成可执行的二进制程序上传到lambda中调用。

2.解决
2.1  ffmpeg

ffmpeg的问题很好解决，在其官网可以下载可执行程序而不必通过install的方式，不再赘述。

2.2  xvfb

查了很多资料，这方面信息不是很多，无奈只好自己编译，过程中发现，除了xvfb之外，还需要编译xkeyboard和xkbcomp。要注意的是，lambda实例中，工作目录是/var/task。

编译xkeyboard

wget https://www.x.org/archive/individual/data/xkeyboard-config/xkeyboard-config-2.19.tar.gz

tar -xzf xkeyboard-config-2.19.tar.gz && cd /var/task/xkeyboard-config-2.19

//设置LD_LIBRARY_PATH（指定动态连接库路径）
export LD_LIBRARY_PATH=/var/task/lib;
export PKG_CONFIG_PATH=/var/task/share/pkgconfig:/var/task/lib/pkgconfig;

//指定输出文件夹,指定xkb输出路径
./configure --prefix=/var/task --with-xkb-base=/var/task/xkb

make && make install

编译xkbcomp

wget https://www.x.org/releases/individual/app/xkbcomp-1.3.1.tar.gz
tar -xzf xkbcomp-1.3.1.tar.gz && cd /app/xkbcomp-1.3.1

./configure --prefix=/var/task --with-xkb-config-root=/var/task/xkb
    
make && make install
编译xvfb

wget https://www.x.org/archive/individual/xserver/xorg-server-1.15.0.tar.gz
tar -xzf xorg-server-1.15.0.tar.gz && cd /app/xorg-server-1.15.0

./configure --prefix=/var/task --with-xkb-path=/var/task/xkb --with-xkb-output=/tmp 

--with-xkb-bin-directory=/var/task/bin

make && make install
这一步会生成许多文件，其中需要用到的，Xvfb（主角，xvfb的可执行版本，它需要依赖xkbcomp和一些动态库），xkbcomp，xkb以及一些动态库。

2.3  firefox，firefox编译起来不太好操作，于是勉强找了一个别人做好的，在容器中调试通过后将一些动态库拷贝了出来。

首先需要准备lambda镜像的firefox相关环境，这样可以得到所有我们缺少的动态库文件。

这里用lambda-linux的epll repository

curl -X GET -o RPM-GPG-KEY-lambda-epll https://lambda-linux.io/RPM-GPG-KEY-lambda-epll

sudo rpm --import RPM-GPG-KEY-lambda-epll

curl -X GET -o epll-release-2017.03-1.2.ll1.noarch.rpm https://lambda-linux.io/epll-release-2017.03-1.2.ll1.noarch.rpm

sudo yum -y install epll-release-2017.03-1.2.ll1.noarch.rpm

//firefox环境
sudo yum --enablerepo=epll install firefox-compat
接着下载firefox免安装版本，此处用45.3，跟高版本环境没找到。

wget -O firefox.tar.bz2 "https://download.mozilla.org/?product=firefox-45.3.0esr-SSL&os=linux64&lang=en-US"

tar -jxvf firefox.tar.bz2
此时在firefox文件夹中的firefox即可以直接运行。

所需全部文件如下：



2.4  main函数

lambda的入口函数，这里用go实现

package main


import (
    "os/exec"
    "context"
    "time"
    // "runtime"
    "strings"
    "bytes"
    "github.com/aws/aws-lambda-go/lambda"  
)

func handler(ctx context.Context) (string, error) {
    
    var str string

    // var args_xvfb []string
    // args_xvfb = append(args_xvfb, strings.Split(":1 -screen 0 800x600x24 &", " ")...)
    c := exec.Command("./Xvfb",":1 -screen 0 1024x768x24")
    var stdout, stderr bytes.Buffer
    c.Stdout = &stdout
    c.Stderr = &stderr
    c.Start()
    outStr, errStr := string(stdout.Bytes()), string(stderr.Bytes())
    str  += "Xvfb start : script Stdout: " + outStr + " --- script Stderr: " + errStr + "****************"

    time.Sleep(5 * time.Second)

    var args_firefox []string
    args_firefox = append(args_firefox, strings.Split(" --display :1 -profile /var/task/firefox_profile -P default", " ")...)
    c_firefox := exec.Command("./firefox/firefox", args_firefox...)
    // c_firefox := exec.Command("./firefox/firefox"," --display :1 -profile /var/task/firefox_profile -P default")
    var stdout_firefox, stderr_firefox bytes.Buffer
    c_firefox.Stdout = &stdout_firefox
    c_firefox.Stderr = &stderr_firefox
    c_firefox.Start()
    outStr_firefox, errStr_firefox := string(stdout_firefox.Bytes()), string(stderr_firefox.Bytes())
    str  += "firefox start : script Stdout: " + outStr_firefox + " --- script Stderr: " + errStr_firefox + "****************"

    time.Sleep(5 * time.Second)

    var args_ffmpeg []string
    args_ffmpeg = append(args_ffmpeg, strings.Split("-y -f x11grab -thread_queue_size 20 -s 1024x768 -framerate 25 -show_region 1 -i :1 -c:v libx264 -f flv rtmp://pili-publish.daishuclass.cn/daishu-video/test_1234", " ")...)
    // //                                 -y -f x11grab -thread_queue_size 20 -s 800x600 -framerate 25 -show_region 1 -i :1 -c:v libx264 -f flv $rtmpUrl
    c_ffmpeg := exec.Command("./ffmpeg", args_ffmpeg...)
    // c_ffmpeg := exec.Command("./ffmpeg -y -f x11grab -thread_queue_size 20 -s 1024x768 -framerate 25 -show_region 1 -i :1 -c:v libx264 -f flv rtmp://pili-publish.daishuclass.cn/daishu-video/test_1234")
    var stdout_ffmpeg, stderr_ffmpeg bytes.Buffer
    c_ffmpeg.Stdout = &stdout_ffmpeg
    c_ffmpeg.Stderr = &stderr_ffmpeg
    c_ffmpeg.Run()
    outStr_ffmpeg, errStr_ffmpeg := string(stdout_ffmpeg.Bytes()), string(stderr_ffmpeg.Bytes())
    str  += "ffmpeg start : script Stdout: " + outStr_ffmpeg + " --- script Stderr: " + errStr_ffmpeg

    return str, nil
}

func main() {
 lambda.Start(handler)
}

以上只是做一个记录，具体lambda的使用可见官网。

吐个槽，目前市场上各种技术，各种云服务层出不穷，好事是好事，但感觉眼花缭乱。虽然用过很多，但总有一种流于表面的不安。

以上。
