����ʹ��java�Ŀ����߿϶�����ʹ��java.\*��javax.\*�Ŀ⡣���ⲻ��SUN��JDKΨһ�ṩ�Ĺ��߿⡣�кܶ�Ĺ��߿���tools.jar�����棬����֮һ����Attach API��ͨ����API������Ҳ���Բ³���API���������ó������attach��Ŀ���JVM�ϡ�ͨ��attach��Ŀ��JVM�ϣ�����Լ�ص�JVM��״̬���ҿ��Դ��з���һЩ�����Ȥ�����⡣Attach API��com.sun.tools.attach �� com.sun.tools.attach.spi�����棬�ܹ���7���๹�ɡ�Ȼ���㲻��Ҫʹ�õ�com.sun.tools.attach.spi ���µ��࣬����Ҫ�˽����VirtualMachine ��VirtualMachineDescriptor �ࡣ
#��ʶAttach API
The VirtualMachine class represents a specific Java virtual machine (JVM) instance. You connect to a JVM by providing the VirtualMachine class with the process id, and then you load a management agent to do your customized behavior:
VirtualMachine ���ʾJVM��ʾ���������ͨ������VirtualMachine��Ŀ��JVM�Ľ���id����������Ŀ��JVM����������Լ���һ��agent������������飺
```java
VirtualMachine vm = VirtualMachine.attach (processid);
String agent = ...
vm.loadAgent(agent);
```
��һ������JVM�ķ�ʽ��ͨ���������е�JVMʾ����ѡ����ϣ�����ӵ�JVM��
```java
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
```
����Ҫ�Ͽ����ӵ�ʱ�򣬿��Ե���detach���������������agent����Ӧ�õ���detach������
##ʾ��
��JDK����Ȼ����һ��JMX������management-agent.jar�����档JMX�������Զ����Ŀ��JVM��������һ��JMX����MBean����Ȼ��õ�һ�����ӵ��˷����MBeanServerConnection��ͨ�������������Լ�ص�JVM�ĺܶ���Ϊ������õ�Զ��Ŀ��JVM���߳�״̬����������ͨ��һ���򵥵�ʾ������ʵ�ִ˹��ܡ����ȣ����ǵĳ�����Ҫattach��һ��Ŀ��JVM�ϡ�Ȼ����Ҫ����һ��Զ�̵�JMX�������û�������Ļ�����������jar��management-agent.jar������Ŀ��JVM�ϵ�java.homeϵͳ�������ҵ���һ�����ӳɹ������ǾͿ��Ի�ȡ�����ӣ���ͨ�������ӻ�ȡ���Ǹ���Ȥ����Ϣ�������̵߳����ֺ�״̬
```java
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
```
JMX�ĸ��������뿴 http://www.ibm.com/developerworks/cn/java/j-lo-jse63/index.html
