---
layout:     post
title:      认识tomcat中的classloader
subtitle:   
date:       2022-02-03
author:     Static
header-img: 
catalog: true
tags:
    - tomcat
    
---

> 本文中介绍的tomcat版本是8.5，代码注释：[https://github.com/whvixd/tomcat](https://github.com/whvixd/tomcat) 分支：8.5.x_local_deploy

# 1. classloader设计图

<html>
    <img src="/img/tomcat/tomcat_classloader.png" width="500" height="300" /> 
</html>

- commonLoader：Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问，加载common/\*中的Java类库；

- catalinaLoader：Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见，加载/server/\*中的Java类库；

- sharedLoader：各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见，加载/shared/\*中的Java类库；

- WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，加载/WebApp/WEB-INF/\*中的Java类库；

> CommonClassLoader、CatalinaClassLoader、SharedClassLoader加载的Java库类在tomcat6之后已经合并到根目录下的lib目录下，见conf/catalina.properties配置文件

# 2. classloader初始化源码

**CommonClassLoader、CatalinaClassLoader、SharedClassLoader 实例化过程：**

链路:Bootstrap#main -> Bootstrap#init -> Bootstrap#initClassLoaders -> Bootstrap#createClassLoader -> ClassLoaderFactory#createClassLoader -> URLClassLoader

```java
// Bootstrap main函数
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



/**
 * Initialize daemon.
 * @throws Exception Fatal initialization error
 */
public void init() throws Exception {
	// whvixd:初始化类加载器
    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

   // whvixd: 当SecurityManager不为空时，加载tomcat相关的类、javax
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

**WebAppClassLoader 实例化过程：**

链路：...->StandardContext#startInternal -> StandardContext#setLoader -> WebappLoader#startInternal -> WebappLoader#createClassLoader -> WebappClassLoaderBase#start


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
    // whvixd:实例化ParallelWebappClassLoader
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

> 所以说tomcat的WebAppClassLoader隔离是通过每个StandardContext维护自己的类加载器，加载自己应用下的`/WEB-INF/classes` 和 `WEB-INF/lib`