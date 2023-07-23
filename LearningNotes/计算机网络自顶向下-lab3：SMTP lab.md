## 实验原理
smtp协议是应用层协议且运行在传输层协议tcp之上，一般默认端口是25号。发送邮件以及接收方接受邮件的具体原理：客户端使用smtp协议将邮件报文推入邮件服务器（我这里是使用的qq邮箱：smtp.qq.com 25）,然后发件方的邮件服务器同样使用smtp协议将邮件报文推入接收方邮件服务器（我这里是使用的网易163邮箱）进入接受方的邮件服务器后，接收方客户需要使用pop3或imap协议从邮件服务器取回邮件。这就是邮件发送与接收的大致流程。接下来我们来看一下具体的实现流程

## python实现细节
- 定义邮件服务器与端口 以及收发邮箱地址
- 创建socket连接到服务器
- 发送HELO命令给服务器（为什么？：HELO命令是SMTP协议的一个重要组成部分，它建立了客户端和服务器之间的联系，并启用了一些额外的功能。因此，在发送任何邮件之前，客户端都需要发送HELO命令以确保SMTP会话的正常启动。--来自chatgpt的解释）
- 发送AUTH LOGIN命令进行身份验证（服务器需要对客户端的身份进行验证）
- 发送mail from rcpt from 对收发方地址进行确认
- 发送具体的message之前还要发送data命令
- 发送具体的message报文，发送完毕后发送quit命令结束
- 以上这些命令都是smtp协议的规范必须遵守才能正确的收发邮件

## python具体实现代码
```python
from socket import *

import ssl

import base64

import logging

  

logging.basicConfig(filename='error.log', level=logging.ERROR)

  

try:

    msg = "\r\n I love computer networks!"

    endmsg = "\r\n.\r\n"

    # Choose a mail server (e.g. Google mail server) and call it mailserver

    mailserver = 'smtp.qq.com'

    # Create socket called clientSocket and establish a TCP connection with mailserver

  
  

    #Fill in start 创建连接

    clientSocket=socket(AF_INET,SOCK_STREAM)

    clientSocket.connect((mailserver,25))

    #ipv4网络 tcp套接字

    #Fill in end 连接建立服务器回复

    recv1 = clientSocket.recv(1024).decode()

    print(recv1)

    if recv1[:3] != '220':

        print('220 reply not received from server.')

    #220回复表示服务器已准备好

    # Send HELO command and print server response.

    heloCommand = 'HELO Alice\r\n'

    clientSocket.send(heloCommand.encode())

    recv2 = clientSocket.recv(1024).decode()

    print(recv2)

    if recv2[:3] != '250':

        print('250 reply not received from server.')

    sender_email='******@qq.com'

    # sender_password='********'

    pwd='******' #qq开通pop/smtp服务授权码 需要登录网页版qq邮箱 使用手机号发送短信获取 具体流程请百度

    sendto_email='******@163.com'

  
  

    # send the AUTH LOGIN command and encode the username and password

    clientSocket.send(b'AUTH LOGIN\r\n')

    response1 = clientSocket.recv(1024)

    print(response1.decode())

    clientSocket.send(base64.b64encode(sender_email.encode()) + b'\r\n')

    response2 = clientSocket.recv(1024)

    print(response2.decode())

    clientSocket.send(base64.b64encode(pwd.encode()) + b'\r\n')

    response3 = clientSocket.recv(1024)

    print(response3.decode())

  

    # Send MAIL FROM command and print server response.

    # Fill in start

    mailfromCommand='mail from:<{}>.\r\n'.format(sender_email)

    clientSocket.send((mailfromCommand.encode()))

    recv3 = clientSocket.recv(1024).decode()

    print(recv3)

    if recv3[:3] !='250' :

        print('250 mail from reply not received from server.')

    # Fill in end

  
  

    # Send RCPT TO command and print server response.

    # Fill in start

    rcptCommand ='rcpt to: <{}>\r\n'.format(sendto_email)

    clientSocket.send((rcptCommand.encode()))

    recv4=clientSocket.recv(1024).decode()

    print(recv4)

    if recv4[:3] !='250' :

        print('250 rcpt to reply not received from server')

    # Fill in end

  
  

    # Send DATA command and print server response.

    # Fill in start

    datacommand='DATA\r\n'

    clientSocket.send((datacommand.encode()))

    recv5=clientSocket.recv(1024).decode()

    print(recv5)

    if recv5[:3] !='354' :

        print('354 data reply not received from server')

    # Fill in end

  
  

    # Send message data.

    # Fill in start

    send ='From: ' +sender_email+ '\r\n'

    send +='To: ' +sendto_email+ '\r\n'

    send +='Subject: ' +'Sending emails with python for the first time!'+ '\r\n\r\n'

    send +=input('Enter your message:\r\n')

    clientSocket.send(send.encode())

    # Fill in end

  

    # Message ends with a single period.

    # Fill in start

    clientSocket.send(endmsg.encode())

    recv6=clientSocket.recv(1024).decode()

    print(recv6)

    if recv6[:3] !='250' :

        print('250 data reply not completely received from server')

    # Fill in end

  
  

    # Send QUIT command and get server response.

    # Fill in start

    quitcommand= 'QUIT\r\n'

    clientSocket.send(quitcommand.encode())

    recv7=clientSocket.recv(1024).decode()

    print(recv7)

    if recv7[:3]!='221':

        print('221 quit reply not received from server')

    # Fill in end

except Exception as e:

     logging.error(str(e), exc_info=True)

clientSocket.close()
```

## 实验截图

![record1_mosaic](https://github.com/Pierceby/first/blob/main/LearningNotes/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%E8%87%AA%E9%A1%B6%E5%90%91%E4%B8%8B-lab3%EF%BC%9ASMTP%20lab.assets/record1_mosaic.png)

## 注意事项以及问题记录
- 出现502 Invalid input from IP address to mail server 错误 可能发送的命令未遵守协议规范，导致命令未激活可以着重检查一下回车与换行是否遵从了规范
- 帐户密码需要使用base64编码
- 如果需要在收件方中显示发件方的信息还要在发送具体message之前，发送发件方信息
- 发送命令要以\r\n结尾 发送消息以'\\r\n.\\r\n'结尾
- 可以导入使用logging库方便查看错误日志

## 参考链接与书籍
- https://juejin.cn/post/6895200594600919048
- https://blog.csdn.net/cunjiu9486/article/details/109071926
- https://blog.csdn.net/qq_37500516/article/details/120050400
- https://github.com/avaiyang/SMTP-Mail-Client/blob/master/smtp_mail_client.py
- https://serversmtp.com/smtp-error/
- [计算机网络自顶向下学生资源](https://media.pearsoncmg.com/aw/ecs_kurose_compnetwork_7/cw/)

