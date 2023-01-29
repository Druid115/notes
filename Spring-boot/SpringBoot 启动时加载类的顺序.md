## SpringBoot 启动时加载外部资源

### 启动器

在 SpringBoot 中，存在 3 种类型的启动器：

- JarLauncher
- WarLauncher
- PropertiesLauncher

当打包为 jar 或 war 时启动器对应前两个。JarLauncher 从 BOOT-INF/lib/ 目录加载 Jar 包，WarLauncher 从 WEB-INF/lib/ 和 WEB-INF/lib-provided/ 加载 Jar 包，如果想添加额外的 Jar 包就需要放在这些目录下。

PropertiesLauncher 默认从 BOOT-INF/lib/ 目录加载 Jar 包，还可以通过 LOADER_PATH 或者 loader.properties 中的 loader.path 配置额外的位置（多个位置逗号隔开）。因此 PropertiesLauncher 主要用于加载外部资源，适用于 SpringBoot thin Jar（瘦身包）的情况。

启动类最终是在打包文件的 MANIFEST.MF 中配置的。



#### 配置使用 PropertiesLauncher

1. Maven 配置：

   ~~~xml
   <build>
       <plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
               <configuration>
                   <mainClass>${start.class}</mainClass>
                   <layout>ZIP</layout>
                   <includes>
                       <include>
                           <groupId>nothing</groupId>
                           <artifactId>nothing</artifactId>
                       </include>
                   </includes>
               </configuration>
               <executions>
                   <execution>
                       <goals>
                           <goal>repackage</goal>
                       </goals>
                   </execution>
               </executions>
           </plugin>
       </plugins>
   </build>
   ~~~

   <layout> 配置可以选择使用哪个启动器，默认根据 <packing> 打包类型确定。

2. 指定其他资源的加载路径：

   ~~~shell
   java -Dloader.path=file:/config -jar xxx.jar
   ~~~



#### 类的加载顺序

PropertiesLauncher 在创建 ClassLoader  时，调用了 `getClassPathArchivesIterator()` 方法，这个方法会获取所有类路径下面的资源文件和 Jar 包：

~~~java
@Override
protected Iterator<Archive> getClassPathArchivesIterator() throws Exception {
    ClassPathArchives classPathArchives = this.classPathArchives;
    if (classPathArchives == null) {
        classPathArchives = new ClassPathArchives();
        this.classPathArchives = classPathArchives;
    }
    return classPathArchives.iterator();
}
~~~

创建了 ClassPathArchives  的单例，在构造方法中：

~~~java
ClassPathArchives() throws Exception {
    this.classPathArchives = new ArrayList<>();
    for (String path : PropertiesLauncher.this.paths) {
        for (Archive archive : getClassPathArchives(path)) {
            debug("paths: " + archive.getUrl());
            addClassPathArchive(archive);
        }
    }
    addNestedEntries();
}
~~~

**这里的 PropertiesLauncher.this.paths 就是通过 loader.path 配置的所有路径，可以看出 loader.path 的优先级更高，因此通过这种方式设置的外部配置文件会优先使用。**

之后的 `addNestedEntries()` 就是加载 Jar 包中的内容：

~~~java
private void addNestedEntries() {
    // The parent archive might have "BOOT-INF/lib/" and "BOOT-INF/classes/"
    // directories, meaning we are running from an executable JAR. We add nested
    // entries from there with low priority (i.e. at end).
    try {
        Iterator<Archive> archives = PropertiesLauncher.this.parent.getNestedArchives(null,
                JarLauncher.NESTED_ARCHIVE_ENTRY_FILTER);
        while (archives.hasNext()) {
            this.classPathArchives.add(archives.next());
        }
    } catch (IOException ex) {
        // Ignore
    }
}
~~~

PropertiesLauncher.this.parent 对应的就是启动的 Jar，这里调用获取嵌套的包，使用 JarLauncher.NESTED_ARCHIVE_ENTRY_FILTER 作为过滤条件：

~~~java
static final EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
    if (entry.isDirectory()) {
        return entry.getName().equals("BOOT-INF/classes/");
    }
    return entry.getName().startsWith("BOOT-INF/lib/");
};
~~~

可以看到当遍历当前 Jar 包时，只会匹配 BOOT-INF/classes/ 目录和 BOOT-INF/lib/ 目里下的所有文件。