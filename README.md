# jmsExample
Objetivo: leer desde una aplicación web Java los mensajes escritos en una cola ActiveMQ

ActiveMQ es una implementación de la especificación JMS dentro del estándar Java EE (https://docs.oracle.com/cd/E19957-01/816-5904-10/816-5904-10.pdf)

Este proyecto se ha generado con el arquetipo maven maven-archetype-webapp desde el IDE eclipse, y al cual se han añadido las siguientes dependencias:
-javax.servlet:servlet-api:2.4 -> simplemente para crear nuestro propio servlet que mapee las peticiones entrantes (No relevante para lecturas desde cola)
-org.springframework:spring-jms:5.3.10 -> La utilidad de Spring que nos permitirá leer los mensajes de la cola
-javax.jms:javax.jms-api:2.0.1 -> Las interfaces de la especificación JMS que serán implementadas por las clases de Spring
-org.apache.activemq:activemq-core:5.7.0 -> Las clases de activemq que emplearemos para crear los recursos de servidor al levantar el Jetty

**PASO 1**: Crear nuestra clase Listener (com.colas.listener.MessageListener). Implementa la interfaz javax.jms.Listener de la especificación JMS la cual tiene un método: onMessage. En nuestro ejemplo simplemente leerá el mensaje y lo pintará por pantalla:

```java
public void onMessage(Message message) {
    if (message instanceof TextMessage) {
        try {
            System.out.println(((TextMessage) message).getText());
        }
        catch (JMSException ex) {
            throw new RuntimeException(ex);
        }
    }
    else {
        throw new IllegalArgumentException("Message must be of type TextMessage");
    }
}
```

**PASO 2**: Editar la configuración de Spring para indicar que será un bean de nuestra clase MessageListener la encargada de leer el mensaje (https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/jms.html):

```xml
<bean id="messageListener" class="com.colas.listener.MessageListener" />  
    
<!-- and this is the message listener container... -->
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="colaTestConnectionFactory"/>
    <property name="destination" ref="colaTestQueue"/>
    <property name="messageListener" ref="messageListener" />
</bean>
```

**PASO 3**: Indicar en el fichero de configuración de jetty (jetty-env.xml) los dos recursos que se van a crear para leer de la cola: La conectionFactory y el acceso a la cola concreta:

```xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">

 <!-- an XADataSource                                                -->
 <New id="myColaTestCF" class="org.eclipse.jetty.plus.jndi.Resource">
   <Arg>jms/CF/colaTest</Arg>
   <Arg>
     <New class="org.apache.activemq.ActiveMQConnectionFactory">
       <Set name="userName">admin</Set>
       <Set name="password">admin</Set>
       <Set name="brokerURL">tcp://localhost:61616</Set>
     </New>
   </Arg>
 </New>


 <New id="myColaTestQ" class="org.eclipse.jetty.plus.jndi.Resource">
   <Arg>jms/Q/colaTest</Arg>
   <Arg>
     <New class="org.apache.activemq.command.ActiveMQQueue">
      <Arg>colaTest</Arg>
     </New>
   </Arg>
 </New>
</Configure>
```

Como se ve en el argumento pasado al constructor de la clase ActiveMQQueu, el nombre de la cola será colaTest. Las propiedades de ActiveMQConnectionFactory (userName, password y brokerURL) vienen con estos valores por defecto al arrancar ActiveMQ

**PASO 4**: Acceder a los recursos del paso 3 mediante jndi desde la configuración de Spring:

```xml
<jee:jndi-lookup id="colaTestConnectionFactory" jndi-name="jms/CF/colaTest"/>
<jee:jndi-lookup id="colaTestQueue" jndi-name="jms/Q/colaTest"/> 
```

Tras estos pasos ya tendríamos la aplicación lista para poder leer los mensajes escritos en la cola, ya sólo tendríamos que instalar ActiveMQ en nuestro equipo. Para ello (en Windows) descargamos el zip de aquí https://activemq.apache.org/components/classic/download/ y lo descomprimimos en cualquier carpeta. Una vez hecho esto nos movemos a la carpeta descargada desde la línea de comandos y desde la carpeta bin arrancamos el servidor de colas con la sentencia activemq start:
![image](https://user-images.githubusercontent.com/32089240/136984638-91fbcf0c-39c1-402f-b8a8-e7dd87aa8f54.png)

Una vez arrancado podemos acceder a la consola de administrador del servidor de colas y crear nuestra cola (http://localhost:8161/admin admin/admin):
![image](https://user-images.githubusercontent.com/32089240/136985118-ebfcdf1e-8d4d-4abc-b3f4-eeff36ab6019.png)

Y creamos nuestra cola:
![image](https://user-images.githubusercontent.com/32089240/136985281-cdff872a-4d2b-418c-a787-e46acc084ddf.png)

Una vez hecho esto arrancamos nuestra aplicación (mvn jetty:run) y escribimos un mensaje desde el administrador de activemq. Si todo funciona correctamente, el mensaje saldrá por la consola de nuestro IDE:
![image](https://user-images.githubusercontent.com/32089240/136985617-48b1df96-3b2b-4ba6-b233-834932338883.png)

Escribimos el mensaje y enviamos:
![image](https://user-images.githubusercontent.com/32089240/136985881-a0402792-6b0e-425b-98ae-de44fd918826.png)

Y vemos como el mensaje ha pasado por nuestra clase y se ha escrito por consola:
![image](https://user-images.githubusercontent.com/32089240/136986384-39b1df7e-02f3-4d91-b2c1-782ca12938ca.png)

