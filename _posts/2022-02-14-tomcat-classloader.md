---
layout:     post
title:      认识Tomcat中的类加载器
subtitle:   
date:       2022-02-03
author:     Static
header-img: 
catalog: true
tags:
    - Tomcat
    
---

> 本文中介绍的Tomcat版本是8.5，代码注释：[https://github.com/whvixd/tomcat](https://github.com/whvixd/tomcat) 分支：8.5.x_local_deploy

# 1. 类加载器设计图

<html>
    <img src="/img/tomcat/tomcat_classloader.png" width="500" height="300" /> 
</html>

- commonLoader：Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问，加载common/\*中的Java类库；

- catalinaLoader：Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见，加载/server/\*中的Java类库；

- sharedLoader：各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见，加载/shared/\*中的Java类库；

- WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，加载/WebApp/WEB-INF/\*中的Java类库；

> CommonClassLoader、CatalinaClassLoader、SharedClassLoader加载的Java库类在Tomcat6之后已经合并到根目录下的lib目录下，见conf/catalina.properties配置文件

# 2. 类加载器初始化过程

**CommonClassLoader、CatalinaClassLoader、SharedClassLoader 实例化过程：**

链路:Bootstrap#main -> Bootstrap#init -> Bootstrap#initClassLoaders -> Bootstrap#createClassLoader -> ClassLoaderFactory#createClassLoader -> URLClassLoader#new

```java
// Bootstrap.java
public static void main(String args[]) {

        synchronized (daemonLock) {
            if (daemon == null) {
                // Don't set daemon until init() has completed
                Bootstrap bootstrap = new Bootstrap();
                try {
                    // whvixd:
                    // 1. this.catalinaDaemon指向实例化的Catalina对象，由CatalinaClassLoader加载
                    // 2. 初始化commonLoader、catalinaLoader、sharedLoader类加载器；commonLoader作为catalinaLoader和sharedLoader的父加载器
                    bootstrap.init();
                } catch (Throwable t) {
                    handleThrowable(t);
                    t.printStackTrace();
                    return;
                }
                daemon = bootstrap;
            } else {
                // When running as a service the call to stop will be on a new
                // thread so make sure the correct class loader is used to
                // prevent a range of class not found exceptions.
                Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
            }
        }
        // ... 省略启动代码 ...
}


// Bootstrap.java
public void init() throws Exception {
	// whvixd:初始化类加载器
    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

   // whvixd: 当SecurityManager不为空时，加载Tomcat相关的类、javax
    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    // whvixd:以catalinaLoader为根，隐性加载;
    // whvixd:当前类由AppClassLoader加载
    Object startupInstance = startupClass.getConstructor().newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    // whvixd:通过反射调用Catalina#setParentClassLoader方法，sharedLoader作为parentClassLoader
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    catalinaDaemon = startupInstance;
}

// Bootstrap.java
private void initClassLoaders() {
    try {
    	// whvixd:创建common类加载器
        commonLoader = createClassLoader("common", null);
        if (commonLoader == null) {
            // no config file, default to this loader - we might be in a 'single' env.
            commonLoader = this.getClass().getClassLoader();
        }
        // whvixd:创建catalina类加载器，并指定父加载器commonLoader
        catalinaLoader = createClassLoader("server", commonLoader);
        // whvixd:创建shared类加载器，并指定父加载器commonLoader
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}

// Bootstrap.java
// whvixd:创建不同的类加载器，加载不同路径下jar
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {

    // whvixd:common.loader:"${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
    String value = CatalinaProperties.getProperty(name + ".loader");
    if ((value == null) || (value.equals("")))
        return parent;

    // whvixd:common 加载 lib中的jar
    value = replace(value);

    List<Repository> repositories = new ArrayList<>();

    String[] repositoryPaths = getPaths(value);

    for (String repository : repositoryPaths) {
        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(new Repository(repository, RepositoryType.DIR));
        }
    }
    // whvixd:通过java.net.URLClassLoader创建类加载器
    return ClassLoaderFactory.createClassLoader(repositories, parent);
}
```

**WebAppClassLoader实例化过程：**

链路：...-> StandardContext#startInternal -> StandardContext#setLoader -> WebappLoader#startInternal -> WebappLoader#createClassLoader -> WebappClassLoaderBase#start


```java
// StandardContext.java
protected synchronized void startInternal() throws LifecycleException {

    if(log.isDebugEnabled())
        log.debug("Starting " + getBaseName());

    // Send j2ee.state.starting notification
    if (this.getObjectName() != null) {
        Notification notification = new Notification("j2ee.state.starting",
                this.getObjectName(), sequenceNumber.getAndIncrement());
        broadcaster.sendNotification(notification);
    }

    setConfigured(false);
    boolean ok = true;

    // Currently this is effectively a NO-OP but needs to be called to
    // ensure the NamingResources follows the correct lifecycle
    if (namingResources != null) {
        namingResources.start();
    }

    // Post work directory
    postWorkDirectory();

    // Add missing components as necessary
    if (getResources() == null) {   // (1) Required by Loader
        if (log.isDebugEnabled())
            log.debug("Configuring default Resources");

        try {
            setResources(new StandardRoot(this));
        } catch (IllegalArgumentException e) {
            log.error(sm.getString("standardContext.resourcesInit"), e);
            ok = false;
        }
    }
    if (ok) {
        resourcesStart();
    }

    if (getLoader() == null) {
        // whvixd:创建WebAppClassLoader，每个StandardContext都会拥有自己的类加载器，这也就是每个应用隔离的原因
        WebappLoader webappLoader = new WebappLoader();
        webappLoader.setDelegate(getDelegate());
        setLoader(webappLoader);
    }
// ...
}

// WebappLoader.java
protected void startInternal() throws LifecycleException {

    if (log.isDebugEnabled())
        log.debug(sm.getString("webappLoader.starting"));

    if (context.getResources() == null) {
        log.info("No resources for " + context);
        setState(LifecycleState.STARTING);
        return;
    }

    // Construct a class loader based on our current repositories list
    try {
		// whvixd:创建WebAppClassLoader
        classLoader = createClassLoader();
        classLoader.setResources(context.getResources());
        classLoader.setDelegate(this.delegate);

        // Configure our repositories
        setClassPath();

        setPermissions();

        ((Lifecycle) classLoader).start();

        String contextName = context.getName();
        if (!contextName.startsWith("/")) {
            contextName = "/" + contextName;
        }
        ObjectName cloname = new ObjectName(context.getDomain() + ":type=" +
                classLoader.getClass().getSimpleName() + ",host=" +
                context.getParent().getName() + ",context=" + contextName);
        Registry.getRegistry(null, null)
            .registerComponent(classLoader, cloname, null);

    } catch (Throwable t) {
        t = ExceptionUtils.unwrapInvocationTargetException(t);
        ExceptionUtils.handleThrowable(t);
        log.error( "LifecycleException ", t );
        throw new LifecycleException("start: ", t);
    }

    setState(LifecycleState.STARTING);
}

// WebappLoader.java
private WebappClassLoaderBase createClassLoader()
    throws Exception {

    Class<?> clazz = Class.forName(loaderClass);
    WebappClassLoaderBase classLoader = null;

    if (parentClassLoader == null) {
        parentClassLoader = context.getParentClassLoader();
    } else {
        context.setParentClassLoader(parentClassLoader);
    }
    Class<?>[] argTypes = { ClassLoader.class };
    Object[] args = { parentClassLoader };
    Constructor<?> constr = clazz.getConstructor(argTypes);
    // whvixd:实例化ParallelWebappClassLoader继承URLClassLoader
    classLoader = (WebappClassLoaderBase) constr.newInstance(args);

    return classLoader;
}

// WebappClassLoaderBase.java 添加WebApp依赖库
public void start() throws LifecycleException {

    state = LifecycleState.STARTING_PREP;

    WebResource[] classesResources = resources.getResources("/WEB-INF/classes");
    for (WebResource classes : classesResources) {
        if (classes.isDirectory() && classes.canRead()) {
            localRepositories.add(classes.getURL());
        }
    }
    WebResource[] jars = resources.listResources("/WEB-INF/lib");
    for (WebResource jar : jars) {
        if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
            localRepositories.add(jar.getURL());
            jarModificationTimes.put(
                    jar.getName(), Long.valueOf(jar.getLastModified()));
        }
    }

    state = LifecycleState.STARTED;
}
```

> 所以说Tomcat的WebAppClassLoader隔离性是通过每个StandardContext维护自己的类加载器，去加载自己应用下的`/WEB-INF/classes` 和 `WEB-INF/lib`中库类

# 3. Tomcat热部署过程

链路：... -> StandardContext#startInternal -> ContainerBase#threadStart -> ContainerBackgroundProcessor#run -> ContainerBase#processChildren -> StandardContext#backgroundProcess -> WebappLoader#backgroundProcess -> StandardContext#reload


```java
// StandardContext.java
protected synchronized void startInternal() throws LifecycleException {
// ...

        // Start ContainerBackgroundProcessor thread
// whvixd:========启动热部署线程，监控是否有class被修改 ========
        super.threadStart();
    } finally {
        // Unbinding thread
        unbindThread(oldCCL);
    }
// ...
}

// ContainerBase.java
protected void threadStart() {

    if (thread != null)
        return;
    if (backgroundProcessorDelay <= 0)
        return;

    threadDone = false;
    String threadName = "ContainerBackgroundProcessor[" + toString() + "]";
    // whvixd:热部署线程
    thread = new Thread(new ContainerBackgroundProcessor(), threadName);
    thread.setDaemon(true);
    // whvixd:作为守护线程启动
    thread.start();

}

// ContainerBackgroundProcessor继承Runnable
@Override
public void run() {
    Throwable t = null;
    String unexpectedDeathMessage = sm.getString(
            "containerBase.backgroundProcess.unexpectedThreadDeath",
            Thread.currentThread().getName());
    try {
        while (!threadDone) {
            try {
                Thread.sleep(backgroundProcessorDelay * 1000L);
            } catch (InterruptedException e) {
                // Ignore
            }
            if (!threadDone) {
            	// whvixd:处理事件
                processChildren(ContainerBase.this);
            }
        }
    } catch (RuntimeException|Error e) {
        t = e;
        throw e;
    } finally {
        if (!threadDone) {
            log.error(unexpectedDeathMessage, t);
        }
    }
}

// ContainerBase.java
protected void processChildren(Container container) {
    ClassLoader originalClassLoader = null;

    try {
        if (container instanceof Context) {
            Loader loader = ((Context) container).getLoader();
            // Loader will be null for FailedContext instances
            if (loader == null) {
                return;
            }

            // Ensure background processing for Contexts and Wrappers
            // is performed under the web app's class loader
            originalClassLoader = ((Context) container).bind(false, null);
        }
        // whvixd:热部署
        container.backgroundProcess();
        Container[] children = container.findChildren();
        for (Container child : children) {
            if (child.getBackgroundProcessorDelay() <= 0) {
                processChildren(child);
            }
        }
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("Exception invoking periodic operation: ", t);
    } finally {
        if (container instanceof Context) {
            ((Context) container).unbind(false, originalClassLoader);
       }
    }
}
}

// WebappLoader.java
public void backgroundProcess() {
    // whvixd:判断class是否有被修改
    if (reloadable && modified()) {
        try {
            Thread.currentThread().setContextClassLoader
                (WebappLoader.class.getClassLoader());
            if (context != null) {
            	// whvixd:重新部署
                context.reload();
            }
        } finally {
            if (context != null && context.getLoader() != null) {
                Thread.currentThread().setContextClassLoader
                    (context.getLoader().getClassLoader());
            }
        }
    }
}

// StandardContext.java
public synchronized void reload() {

    // whvixd:启动一个守护线程，轮训校验是否有class文件被修改，若有修改就停止后再启动，重新加载新的class

    // Validate our current component state
    if (!getState().isAvailable())
        throw new IllegalStateException
            (sm.getString("standardContext.notStarted", getName()));

    if(log.isInfoEnabled())
        log.info(sm.getString("standardContext.reloadingStarted",
                getName()));

    // Stop accepting requests temporarily.
    setPaused(true);

    try {
        // whvixd:发送停止事件
        stop();
    } catch (LifecycleException e) {
        log.error(
            sm.getString("standardContext.stoppingContext", getName()), e);
    }

    try {
        // whvixd:发送启动事件，重新加载class
        start();
    } catch (LifecycleException e) {
        log.error(
            sm.getString("standardContext.startingContext", getName()), e);
    }

    setPaused(false);

    if(log.isInfoEnabled())
        log.info(sm.getString("standardContext.reloadingCompleted",
                getName()));

}
```
> Tomcat的热部署现实就是启动一个守护线程，轮训校验是否有class文件被修改，若有修改就停止后再启动，重新加载新的class