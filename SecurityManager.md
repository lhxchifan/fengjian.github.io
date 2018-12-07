# SecurityManager

## 用途

> 1. 用于实现一些安全策略，在应用程序中动态得接入，并在配置文件中保存一些策略
>
> 2. 可以处理以下的permission
>
>    ```java
>    java.io.FilePermission, java.net.SocketPermission, java.net.NetPermission, java.security.SecurityPermission, java.lang.RuntimePermission, java.util.PropertyPermission, java.awt.AWTPermission, java.lang.reflect.ReflectPermission, and java.io.SerializablePermission
>    ```
>
> 3. 可以定义一些是否有权限读写文件，连接某地址某端口，能否stopThread等

## 用法

> 1. 声明方式
>
>    ```
>    java -Djava.security.manager <class_name>
>    or 
>    SecurityManager sm = new SecurityManager();
>    System.setSecurityManager(sm);
>    ```
>
> 2. 默认开启安全管理类后，JVM会先去 `${java.home}/jre/lib/security` 下找到 `java.security` 获取到其中的一段配置
>
>    ```java
>    # The default is to have a single system-wide policy file,
>    # and a policy file in the user's home directory.
>    policy.url.1=file:${java.home}/lib/security/java.policy
>    policy.url.2=file:${user.home}/.java.policy
>    ```
>
> 3. 这段配置就是policy文件的目标位置，默认位置为 `${java.home}/lib/security/java.policy`
>
> 4. 这个文件的内容
>
>    ```java
>    // Standard extensions get all permissions by default
>    
>    grant codeBase "file:${{java.ext.dirs}}/*" {
>            permission java.security.AllPermission;
>    };
>    
>    // default permissions granted to all domains
>    
>    grant {
>            // Allows any thread to stop itself using the java.lang.Thread.stop()
>            // method that takes no argument.
>            // Note that this permission is granted by default only to remain
>            // backwards compatible.
>            // It is strongly recommended that you either remove this permission
>            // from this policy file or further restrict it to code sources
>            // that you specify, because Thread.stop() is potentially unsafe.
>            // See the API specification of java.lang.Thread.stop() for more
>            // information.
>            permission java.lang.RuntimePermission "stopThread";
>    
>            // allows anyone to listen on dynamic ports
>            permission java.net.SocketPermission "localhost:0", "listen";
>    
>            // "standard" properies that can be read by anyone
>    
>            permission java.util.PropertyPermission "java.version", "read";
>            permission java.util.PropertyPermission "java.vendor", "read";
>            permission java.util.PropertyPermission "java.vendor.url", "read";
>            permission java.util.PropertyPermission "java.class.version", "read";
>            permission java.util.PropertyPermission "os.name", "read";
>            permission java.util.PropertyPermission "os.version", "read";
>            permission java.util.PropertyPermission "os.arch", "read";
>            permission java.util.PropertyPermission "file.separator", "read";
>            permission java.util.PropertyPermission "path.separator", "read";
>            permission java.util.PropertyPermission "line.separator", "read";
>    
>            permission java.util.PropertyPermission "java.specification.version", "read";
>            permission java.util.PropertyPermission "java.specification.vendor", "read";
>            permission java.util.PropertyPermission "java.specification.name", "read";
>    
>            permission java.util.PropertyPermission "java.vm.specification.version", "read";
>            permission java.util.PropertyPermission "java.vm.specification.vendor", "read";
>            permission java.util.PropertyPermission "java.vm.specification.name", "read";
>            permission java.util.PropertyPermission "java.vm.version", "read";
>            permission java.util.PropertyPermission "java.vm.vendor", "read";
>            permission java.util.PropertyPermission "java.vm.name", "read";
>    };
>    ```
>
> 5. `permission java.lang.RuntimePermission "stopThread";`这里允许了调用Thread的stop方法