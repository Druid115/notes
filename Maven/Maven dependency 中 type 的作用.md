### Maven dependency 中 type 的作用

dependency 中 type 默认为 jar，即引入一个特定的 ja r包。

当我们需要引入很多 jar 包的时候会导致 pom.xml 过大，我们可以想到的一种解决方案是定义一个父项目，但是父项目只有一个，也有可能导致父项目的 pom.xml 文件过大。这个时候我们引进来一个 type 为 pom 的 dependencies，意味着我们可以将所有的 jar 包打包成一个 pom，然后我们依赖了 pom，即可以下载下来所有依赖的 jar 包。