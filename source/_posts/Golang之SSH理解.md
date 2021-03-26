---
title: Golang之SSH理解
catalog: true
date: 2021-03-26 17:06:19
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- golang
catagories:
- golang 
updateDate: 2021-03-26 17:06:19
---

以前写过Golang通过SSH执行交换机操作，但是对于证书认证这一块没有深究。这次通过读gopkg文件，理解更深了一步。

### 代码案例

```
package main

import (
	"bytes"
	"fmt"
	"golang.org/x/crypto/ssh"
	"io/ioutil"
	"log"
)

func main() {
	//hnowhost文件对应/root/.ssh/known_hosts。
	var knowhost = []byte("192.168.14.137 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItb...gPfaynABbA/tD1V9pV5w=")
	//只关注pubkey解析与否
	_, _, pubKey, _, _, err := ssh.ParseKnownHosts(knowhost)
	if err != nil {
		log.Fatalf("parseKnowHost error", err)
		return
	}
	//fmt.Println(pubKey)
	//读取本机的私钥
	key, err := ioutil.ReadFile("./gotest/insert/id_rsa_2048")
	if err != nil {
		log.Fatalf("unable to read private key: %v", err)
	}
	//获取签名
	signer, err := ssh.ParsePrivateKey(key)
	if err != nil {
		log.Fatalf("unable to parse private key: %v", err)
	}
	//设置配置文件
	config := &ssh.ClientConfig{
		User: "root",
		Auth: []ssh.AuthMethod{
			//证书验证
			ssh.PublicKeys(signer),
			//密码验证
			//ssh.Password("xxxx"),
		},
		//用于加密期间握手验证主机秘钥的回调函数。就比如第一次连接到一台主机，除了验证，还会弹出一堆信息（让你接受对方公钥）。
		// 用于接收特定主机的秘钥
		HostKeyCallback: ssh.FixedHostKey(pubKey),
		//用于接收任何主机的秘钥
		//HostKeyCallback: ssh.InsecureIgnoreHostKey(),
	}
	//准备建立客户端连接
	client, err := ssh.Dial("tcp", "192.168.14.137:22", config)
	if err != nil {
		log.Fatalf("unable to connect: %v", err)
	}
	defer client.Close()
	//一个正式用于执行远程命令或者shell的连接
	session, err := client.NewSession()
	if err != nil {
		log.Fatalf("Failed to create session: ", err)
	}
	defer session.Close()
	var b bytes.Buffer
	session.Stdout = &b
	if err := session.Run("/usr/bin/whoami"); err != nil {
		log.Fatal("Failed to run: " + err.Error())

	}
	fmt.Println(b.String())

}

```



### [ParseKnownHosts](https://github.com/golang/crypto/blob/0c34fe9e7dc2/ssh/keys.go#L123) 

​	为什么要这个函数，因为我们要通过这个函数获取公钥，用于下面的HostKeyCallBack.

​	这里对应的是linux中的/root/.ssh/known_hosts（只有建立过连接的才有哦）。详细说明请查找man sshd。

​	know_hosts的作用：ssh会把你每个你访问过计算机的公钥(public key)都记录在~/.ssh/known_hosts。当下次访问相同计算机时，OpenSSH会核对公钥。如果公钥不同，OpenSSH会发出警告， 避免你受到DNS Hijack之类的攻击。

```
名称  算法  公钥
192.168.14.137 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPQphM3fQtoe5vq//ROvwfAI6aUAt8VtNzj/GdGCeYU1VJMrWrS/wbwnUYZ3DPVEG7wgPfaynABbA/tD1V9pV5w=
```

​     

​	怎么获取这种文件呢：这里提供三种方式：

​    1. 通过官方库读取本地的know_host文件。（golang.org/x/crypto/ssh/knownhosts）

```
hostKeyCallback, err := knownhosts.New("/Users/user/.ssh/known_hosts")
```

​	参考链接： [key Verification](https://skarlso.github.io/2019/02/17/go-ssh-with-host-key-verification/)

 2. 自己写程序获取know_host文件

    ```
    func getHostKey(host string) (ssh.PublicKey, error) {
        file, err := os.Open(filepath.Join(os.Getenv("HOME"), ".ssh", "known_hosts"))
        if err != nil {
            return nil, err
        }
        defer file.Close()
    
        scanner := bufio.NewScanner(file)
        var hostKey ssh.PublicKey
        for scanner.Scan() {
            fields := strings.Split(scanner.Text(), " ")
            if len(fields) != 3 {
                continue
            }
            if strings.Contains(fields[0], host) {
                var err error
                hostKey, _, _, _, err = ssh.ParseAuthorizedKey(scanner.Bytes())
                if err != nil {
                    return nil, errors.New(fmt.Sprintf("error parsing %q: %v", fields[2], err))
                }
                break
            }
        }
    
        if hostKey == nil {
            return nil, errors.New(fmt.Sprintf("no hostkey for %s", host))
        }
        return hostKey, nil
    }
    ```

    参考链接：[ssh handshake](https://stackoverflow.com/questions/45441735/ssh-handshake-complains-about-missing-host-key/45442842)

 3.  按照know_hosts格式，手动写一行数据，参照第一行代码。

    使用ssh-keyscan获取know_host信息，建议-t 写成ecdsa。rsa好像没有生效。

    ```
    ssh-keyscan -v  -t ecdsa 192.168.14.137
    192.168.14.137 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItb...gPfaynABbA/tD1V9pV5w=
    --- 
    var knowhost = []byte("192.168.14.137 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPQphM3fQtoe5vq//ROvwfAI6aUAt8VtNzj/GdGCeYU1VJMrWrS/wbwnUYZ3DPVEG7wgPfaynABbA/tD1V9pV5w=")
    	//只关注pubkey解析与否
    	_, _, pubKey, _, _, err := ssh.ParseKnownHosts(knowhost)
    ```

    参考链接： [ssh-keyscan命令详解](https://ghostwritten.blog.csdn.net/article/details/113554538)

    

### [HostKeyCallback](https://github.com/golang/crypto/blob/0c34fe9e7dc2/ssh/client.go#L189)

```
type HostKeyCallback func(hostname string, remote net.Addr, key PublicKey) error
```

HostKeyCallback是用于验证服务器密钥的函数类型。

FixedHostKey(pubKey)，用于接收特定主机的秘钥，用于生产。
InsecureIgnoreHostKey() 用于接收任何主机的秘钥。

当然也可以自定义实现类型： 比如上例。

```
func New(files ...string) (ssh.HostKeyCallback, error)
```

参考链接： [knownhosts](https://pkg.go.dev/golang.org/x/crypto/ssh/knownhosts)



### [ParsePrivateKey](https://github.com/golang/crypto/blob/0c34fe9e7dc2/ssh/keys.go#L1076)

```
signer, err := ssh.ParsePrivateKey(key)
config := &ssh.ClientConfig{
		User: "root",
		Auth: []ssh.AuthMethod{
			//证书验证
			ssh.PublicKeys(signer),
			//密码验证
			//ssh.Password("xxxx"),
		},
```

ParsePrivateKey从PEM编码的私钥返回签名者。 它支持与ParseRawPrivateKey相同的键。 如果私钥已加密，它将返回PassphraseMissingError。

主要是免密登录中的秘钥认证。举例

1、登录A机器
2、ssh-keygen -t [rsa|dsa]，将会生成密钥文件和私钥文件 id_rsa,id_rsa.pub或id_dsa,id_dsa.pub
3、将 .pub 文件复制到B机器的 .ssh 目录， 并 cat id_dsa.pub >> ~/.ssh/authorized_keys
4、大功告成，从A机器登录B机器的目标账户，不再需要密码了；（直接运行 #ssh 192.168.20.60 ）



简单来说： 当你将你的公钥文件拷贝到对方主机的/root/.ssh/authorized_keys中去，那么你下次登录时候，选择证书认证的时候就不要输入密码了。

参考链接： [ssh的两种认证方式](https://blog.csdn.net/marywang56/article/details/83621738)







### 参考链接：

https://pkg.go.dev/golang.org/x/crypto/ssh#HostKeyCallback

[ParseKnownHosts](https://github.com/golang/crypto/blob/0c34fe9e7dc2/ssh/keys.go#L123) 

[knownhosts](https://pkg.go.dev/golang.org/x/crypto/ssh/knownhosts)

[HostKeyCallback](https://github.com/golang/crypto/blob/0c34fe9e7dc2/ssh/client.go#L189)

[key Verification](https://skarlso.github.io/2019/02/17/go-ssh-with-host-key-verification/)

[ssh handshake](https://stackoverflow.com/questions/45441735/ssh-handshake-complains-about-missing-host-key/45442842)

[ssh-keyscan命令详解](https://ghostwritten.blog.csdn.net/article/details/113554538)

[knownhosts](https://pkg.go.dev/golang.org/x/crypto/ssh/knownhosts)

[ssh的两种认证方式](https://blog.csdn.net/marywang56/article/details/83621738)

[Validating host keys in Go's ssh package](