---
layout: post
category : [JVM]
tagline: "Supporting tagline"
tags : [Attach API]
---
{% include JB/setup %}


相信使用java的开发者肯定经常使用java.\*和javax.\*的库。但这不是SUN的JDK唯一提供的工具库。有很多的工具库在tools.jar包下面，其中之一就是Attach API。通过此API的名称也可以猜出此API的作用是让程序可以attach到目标的JVM上。通过attach到目标JVM上，你可以监控到JVM的状态并且可以从中发现一些你感兴趣的问题。Attach API在com.sun.tools.attach 和 com.sun.tools.attach.spi包下面，总共由7个类构成。然后你不需要使用到com.sun.tools.attach.spi 包下的类，你需要了解的是VirtualMachine 和VirtualMachineDescriptor 类。

#初识Attach API

VirtualMachine 类表示JVM的示例。你可以通过传给VirtualMachine类目标JVM的进程id参数来连接目标JVM，并且你可以加载一个agent来做额外的事情：

        VirtualMachine vm = VirtualMachine.attach (processid);
        String agent = ...
        vm.loadAgent(agent);

另一种连接JVM的方式是通过迭代所有的JVM示例来选择你希望连接的JVM：

        String name = ...
        List vms = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd: vms) {
            if (vmd.displayName().equals(name)) {
                VirtualMachine vm = VirtualMachine.attach(vmd.id());
                String agent = ...
                vm.loadAgent(agent);
                // ...
            }
        }

当需要断开连接的时候，可以调用detach方法。当你加载完agent后，你应该调用detach方法。

##示例

在JDK中天然存在一个JMX代理，在management-agent.jar包下面。JMX代理可以远程在目标JVM上启动的一个JMX代理MBean服务，然后得到一个连接到此服务的MBeanServerConnection。通过这个连接你可以监控到JVM的很多行为，比如得到远程目标JVM的线程状态。下面我们通过一个简单的示例程序实现此功能。首先，我们的程序需要attach到一个目标JVM上。然后需要启动一个远程的JMX服务（如果没有启动的话）。代理类jar包management-agent.jar可以在目标JVM上的java.home系统环境下找到。一旦连接成功，我们就可以获取此连接，并通过此连接获取我们感兴趣的信息，比如线程的名字和状态


        import java.io.File;
        import java.lang.management.ManagementFactory;
        import java.lang.management.ThreadInfo;
        import java.lang.management.ThreadMXBean;
        import java.util.Set;
        import javax.management.MBeanServerConnection;
        import javax.management.ObjectName;
        import javax.management.remote.JMXConnector;
        import javax.management.remote.JMXConnectorFactory;
        import javax.management.remote.JMXServiceURL;
        import com.sun.tools.attach.VirtualMachine;

        public class MonitorVMThread {
	        public static void main(String args[]) throws Exception {
	            if (args.length != 1) {
	              System.err.println("Please provide process id");
	              System.exit(-1);
	            }
	            VirtualMachine vm = VirtualMachine.attach(args[0]);
	            String connectorAddr = vm.getAgentProperties().getProperty(
	              "com.sun.management.jmxremote.localConnectorAddress");
	            if (connectorAddr == null) {
	              String agent = vm.getSystemProperties().getProperty(
	                "java.home")+File.separator+"lib"+File.separator+
	                "management-agent.jar";
	              vm.loadAgent(agent);
	              connectorAddr = vm.getAgentProperties().getProperty(
	                "com.sun.management.jmxremote.localConnectorAddress");
	              vm.detach();
	            }
	            JMXServiceURL serviceURL = new JMXServiceURL(connectorAddr);
	            JMXConnector connector = JMXConnectorFactory.connect(serviceURL); 
	            MBeanServerConnection mbsc = connector.getMBeanServerConnection(); 
	            ObjectName objName = new ObjectName(
	              ManagementFactory.THREAD_MXBEAN_NAME);
	            Set<ObjectName> mbeans = mbsc.queryNames(objName, null);
	            for (ObjectName name: mbeans) {
	              ThreadMXBean threadBean;
	              threadBean = ManagementFactory.newPlatformMXBeanProxy(
	                mbsc, name.toString(), ThreadMXBean.class);
	              long threadIds[] = threadBean.getAllThreadIds();
	              for (long threadId: threadIds) {
	                ThreadInfo threadInfo = threadBean.getThreadInfo(threadId);
	                System.out.println (threadInfo.getThreadName() + " / " +
	                    threadInfo.getThreadState());
	              }
	            }
	          }
        }


JMX的更多内容请看 http://www.ibm.com/developerworks/cn/java/j-lo-jse63/index.html
