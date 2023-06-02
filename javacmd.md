---
title: java命令执行
date: 2023-6-2 10-13-00
tags: 
  - java代码审计
  - 命令执行
categories:
  - java代码审计
---
##java代码审计之命令执行的多种方式
###三种基本执行方式
```
java.lang.Runtime#exec()     
java.lang.ProcessBuilder#start()
java.lang.ProcessImpl#start()
```
####Runtime  基本执行方式
```
Process Runexec = Runtime.getRuntime().exec("cmd /c dir");  #通过调用Runtime类的静态方法getRuntime返回的runtime类的方法exec来执行命令
InputStream in = Runexec.getInputStream();#返回的数据流要经过手动输出，才能出现在终端中
ByteArrayOutputStream bos = new ByteArrayOutputStream();
byte[] bytes = new byte[1024];
int size;
while((size = in.read(bytes)) > 0)
    bos.write(bytes,0,size);
System.out.println(bos.toString());
```
runtime.exec 调用  
```
return new ProcessBuilder(cmdarray)
            .environment(envp)
            .directory(dir)
            .start();
```
分析 ProcessBuilder start方法  大致分为三块
```
public Process start() throws IOException {
        
    SecurityManager security = System.getSecurityManager();    #存在一个安全性检查，只有配置了安全策略才会生效
        if (security != null)
            security.checkExec(prog);

        String dir = directory == null ? null : directory.toString();

        for (int i = 1; i < cmdarray.length; i++) {  #不允许有\x0000字节的存在，只能有\x00
            if (cmdarray[i].indexOf('\u0000') >= 0) {
                throw new IOException("invalid null character in command");
            }
        }

        
        return ProcessImpl.start(cmdarray,       #调用ProcessImpl.start()方法执行命令
                                     environment,
                                     dir,
                                     redirects,
                                     redirectErrorStream);
    }
```
ProcessImpl start方法构造新的类ProcessImpl
```
static Process start(String cmdarray[],
                         java.util.Map<String,String> environment,
                         String dir,
                         ProcessBuilder.Redirect[] redirects,
                         boolean redirectErrorStream)
                    return new ProcessImpl(cmdarray, envblock, dir,
                                   stdHandles, redirectErrorStream);
                     
```
本质上三种方法都是在调用 ProcessImpl.start()
模拟调用
```
Process Runexec = Runtime.getRuntime().exec("calc");  #成功执行
Process p = new ProcessBuilder(new String[]{"cmd","/c","dir"}).enviroment(null).directory(null).start();    #enviroment方法时内部方法，无法外部调用
去掉enviroment，执行成功
Process p = new ProcessBuilder(new String[]{"cmd","/c","dir"}).directory(null).start();

Process p = ProcessPml.start(new String[]{"cmd","/c","dir"},null,null,null,null,null);    #ProcessPml.start 该方法作用域为defualt无法外部调用
```

作用域调用失败，可以参考下一篇java 反射的原理获取方法，执行命令

