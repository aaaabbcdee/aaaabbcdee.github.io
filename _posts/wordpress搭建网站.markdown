---
layout:     post
title:      "wordpress搭建网站"
subtitle:   "学习记录"
date:       2025-04-19 15:30:00
author:     "bel"
header-img: "img/post-bg-2015.jpg"
tags:
    - study
---

#服务器 #IP地址 #域名 #DNS #ssll证书配置
### 使用wordpress搭建网站的整个过程：

#### 准备工作

服务器购买，配置选择。（操作系统，内存，云盘等）（这里选用AWS，ec2免费套餐，ubuntu系统，30G)
[AWS新手必看！免费层坑多](https://mbd.baidu.com/newspage/data/dtlandingsuper?nid=dt_5170666495418303053&sourceFrom=search_a)

域名购买，托管(cloutflare)，DNS解析到正常访问。 

DNS解析重点：（理解原理）

1. 配置www和ip地址。（通过www.×××.com访问)
2. 配置@和ip地址。通过×××.com访问)
3. 配置CNAME（也称为规范名称记录）：（用于将一个域名指向另一个域名，而不是直接指向IP地址）

（阿里云服务器域名没有进行备案无法访问。）

#### 从零开始从亚马逊云ec2实例安装并使用wordpress（通过AI的协助）

在Ubuntu 24.04上安装和配置WordPress的过程涉及几个步骤，包括准备环境、安装必要的软件（如Apache、MySQL、PHP）、下载并配置WordPress等。以下是详细的步骤指南：

**准备工作**

1. **更新系统**：
   确保您的系统是最新的。

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

**安装Apache**

2. **安装Apache**：
   如果还没有安装Apache，请执行以下命令进行安装。

   ```bash
   sudo apt install apache2 -y
   ```

3. **启动并启用Apache服务**：

   ```bash
   sudo systemctl enable apache2
   sudo systemctl start apache2
   ```

**安装MySQL**

4. **安装MySQL**：
   WordPress需要一个数据库管理系统来存储数据，这里我们使用MySQL。

   ```bash
   sudo apt install mysql-server -y
   ```

5. **启动并启用MySQL服务**：

   ```bash
   sudo systemctl enable mysql
   sudo systemctl start mysql
   ```

6. **设置MySQL root用户密码并完成安全配置**：
   运行MySQL的安全脚本以移除一些不安全的默认设置。

   ```bash
   sudo mysql_secure_installation
   ```

   按照提示操作，为root用户设置一个强密码，并回答一系列关于安全选项的问题。

7. **创建WordPress数据库和用户**：
   登录到MySQL。

   ```bash
   sudo mysql -u root -p
   ```

   在MySQL shell中运行以下SQL语句来创建数据库和用户。

   ```sql
   CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'yourpassword';
   GRANT ALL ON wordpress.* TO 'wordpressuser'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```

**安装PHP**

8. **安装PHP及相关模块**：
   WordPress需要PHP来处理动态内容。

   ```bash
   sudo apt install php libapache2-mod-php php-mysql -y
   ```

9. **重启Apache服务**：
   安装完PHP后，重启Apache使更改生效。

   ```bash
   sudo systemctl restart apache2
   ```

**下载并配置WordPress**

10. **下载WordPress**：
    使用curl或wget从WordPress官方网站下载最新版本。

    ```bash
    cd /tmp
    curl -O https://wordpress.org/latest.tar.gz
    tar xzvf latest.tar.gz
    touch /tmp/wordpress/.htaccess
    cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php
    mkdir /tmp/wordpress/wp-content/upgrade
    ```

11. **复制文件到Web根目录**：
    将解压后的WordPress文件复制到Apache的默认文档根目录下。

    ```bash
    sudo cp -a /tmp/wordpress/. /var/www/html/wordpress
    ```

12. **设置正确的权限**：
    设置正确的文件权限以便Apache可以读写必要文件。

    ```bash
    sudo chown -R www-data:www-data /var/www/html/wordpress
    sudo find /var/www/html/wordpress/ -type d -exec chmod 755 {} \;
    sudo find /var/www/html/wordpress/ -type f -exec chmod 644 {} \;
    sudo chmod 600 /var/www/html/wordpress/wp-config.php
    ```

13. **配置WordPress**：
    编辑`wp-config.php`文件，填入之前创建的数据库信息。

    ```bash
    sudo nano /var/www/html/wordpress/wp-config.php
    ```

    找到以下部分并替换为您自己的值：

    ```php
    define('DB_NAME', 'wordpress');
    define('DB_USER', 'wordpressuser');
    define('DB_PASSWORD', 'yourpassword');
    define('DB_HOST', 'localhost');
    ```

**配置Apache虚拟主机（可选）**(最好配置)

14. **配置虚拟主机**：
    创建一个新的Apache配置文件用于WordPress站点。

    ```bash
    sudo nano /etc/apache2/sites-available/wordpress.conf
    ```

    添加如下内容：

    ```apache
    <VirtualHost *:80>
        ServerAdmin admin@example.com
        DocumentRoot /var/www/html/wordpress
        #使用ip地址时注释这两行
        #ServerName example.com
        #ServerAlias www.example.com
    
        <Directory /var/www/html/wordpress/>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
        </Directory>
    
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```

15. **启用站点和重写模块**：

    ```bash
    sudo a2ensite wordpress.conf
    sudo a2enmod rewrite
    sudo systemctl restart apache2
    ```

16. **禁用默认站点配置（`000-default.conf`）**

    ```
    sudo a2dissite 000-default.con
    ```

17. **配置.htaccess文件**

    1. **检查是否已存在 `.htaccess` 文件**

    首先，检查您的WordPress安装目录（例如 `/var/www/html/wordpress`）中是否已经存在 `.htaccess` 文件。

    ```
    ls -la /var/www/html/wordpress/
    ```

    如果看到 `.htaccess` 文件，那么可以跳到下一步进行验证或编辑。如果没有看到，则需要创建一个新的 `.htaccess` 文件。

    2. **创建或编辑 `.htaccess` 文件**

    **如果不存在 `.htaccess` 文件：**

    使用文本编辑器创建一个新的 `.htaccess` 文件。这里以 `nano` 为例：

    ```
    sudo nano /var/www/html/wordpress/.htaccess
    ```

    然后将以下基本的 WordPress 重写规则粘贴进去：

    ```
    # BEGIN WordPress
    <IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /
    RewriteRule ^index\.php$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule . /index.php [L]
    </IfModule>
    # END WordPress
    ```

    **如果已存在 `.htaccess` 文件：**

    如果您发现已经有 `.htaccess` 文件，请使用文本编辑器打开它并确保其包含上述的重写规则。同样地，您可以使用 `nano` 或您喜欢的任何文本编辑器来编辑这个文件：

    ```
    sudo nano /var/www/html/wordpress/.htaccess
    ```

    3. **设置正确的权限**

    确保 `.htaccess` 文件具有适当的权限，以便 Apache 能够读取它，但同时保护它不被未经授权的用户修改。

    ```
    sudo chown www-data:www-data /var/www/html/wordpress/.htaccess
    sudo chmod 644 /var/www/html/wordpress/.htaccess
    ```

    4. **确认 `mod_rewrite` 已启用**

    WordPress 的 URL 重写功能依赖于 Apache 的 `mod_rewrite` 模块。确保该模块已被启用：

    ```
    sudo a2enmod rewrite
    ```

    如果提示模块已启用，则无需进一步操作。否则，启用后需要重启 Apache 服务使更改生效。

    5. ##### 重启 Apache 服务

    完成 `.htaccess` 文件的创建或修改以及确认 `mod_rewrite` 模块已启用之后，重启 Apache 服务以应用所有更改：

    ```
    sudo systemctl restart apache2
    ```

18. **访问WordPress**：
    在浏览器中输入您的服务器IP地址或域名（例如：`http://example.com`），您应该能看到WordPress的安装向导页面。按照指示完成安装过程，包括选择语言、填写站点标题、创建管理员账户等步骤。

通过以上步骤，您应该能够在Ubuntu 24.04上成功安装并配置WordPress。如果遇到任何问题，请检查Apache和PHP的错误日志获取更多信息。

19. 解决wordpress上传主题时出现的错误：上传的文件大小超过 php.ini 文件中定义的 upload_max_filesize 值

通过修改.htaccess文件：

**步骤 1: 确认 Apache 已启用 `.htaccess` 支持**

首先，请确保您的Apache配置允许在目录中使用 `.htaccess` 文件。这通常涉及到确认虚拟主机或主配置文件中的 `<Directory>` 块内有适当的 `AllowOverride` 设置。

例如，在您的虚拟主机配置文件（可能是 `/etc/apache2/sites-available/wordpress.conf` 或类似的文件）中应该包含如下设置：

```
<Directory /var/www/html/wordpress/>
    Options Indexes FollowSymLinks
    AllowOverride All  # 确保这里设置为 All
    Require all granted
</Directory>
```

如果需要更改此设置，请确保重启Apache服务以应用更改：

```
sudo systemctl restart apache2
```

**步骤 2: 编辑或创建 `.htaccess` 文件**

接下来，您需要编辑或创建位于WordPress根目录下的 `.htaccess` 文件（通常是 `/var/www/html/wordpress/.htaccess`）。您可以使用任何文本编辑器进行操作，比如 `nano`：

```
sudo nano /var/www/html/wordpress/.htaccess
```

在文件中添加或修改以下内容来增加上传文件大小限制：（注意不要放到最后面，不要被包含住）

```
# 设置 PHP 的 upload_max_filesize 和 post_max_size
php_value upload_max_filesize 64M
php_value post_max_size 64M
```

根据您的需求调整这些值。例如，如果您希望允许更大的文件上传，可以将这些值设置得更高，如 `128M` 或更大。

**步骤 3: 保存并退出**

完成编辑后，保存文件并退出编辑器。如果您使用的是 `nano`，可以通过按 `Ctrl+O` 来保存更改，然后按 `Ctrl+X` 退出。

**步骤 4: 测试更改**

现在，尝试再次上传主题或插件到WordPress，检查是否解决了文件大小限制的问题。

**注意事项**

- **权限**：确保 `.htaccess` 文件具有正确的权限，以便Web服务器能够读取它。通常，这意味着该文件应由Web服务器用户（如 `www-data`）拥有，并且权限设置为 `644`。

  ```
  sudo chown www-data:www-data /var/www/html/wordpress/.htaccess
  sudo chmod 644 /var/www/html/wordpress/.htaccess
  ```

- **兼容性检查**：并不是所有的PHP安装都支持通过 `.htaccess` 文件来修改PHP设置。如果您发现这种方法不起作用，可能是因为您的PHP是以CGI/FastCGI模式运行的。在这种情况下，您可能需要考虑其他方法，比如使用用户级的 `php.ini` 文件（`.user.ini`），或者联系您的托管服务提供商寻求帮助。

通过上述步骤，您应该能够在不直接修改 `php.ini` 文件的情况下调整PHP设置，从而解决WordPress上传文件时遇到的大小限制问题。如果仍然遇到问题，请检查Apache和PHP的错误日志以获取更多信息。

20. 总结：需要从中学习到如何来去给一个应用安装对应的环境，如何查找排除错误，如何修改错误以及如何更快的找到解决方案。要学会更好的提示ai来帮助我们快速纠错。了解整个项目部署的过程。



#### 配置服务器允许root+密码登录

[美国AWS EC2 ubuntu 使用密码登陆_aws ec2设置ssh帐号密码登陆-CSDN博客](https://blog.csdn.net/qq_19897551/article/details/143885110)

动态ip地址如何永久配置dns解析。



#### cloudflare配置ssl证书

[告别证书频繁续签！Cloudflare 15年免费SSL证书申请配置图文教程 - 搬主题](https://www.banzhuti.com/free-cloudflare-ssl.html)



#### 亚马逊（aws）服务器上配置ssl证书联系服务器

**第一步：准备阶段**

1. 确保您拥有必要的权限：在AWS中创建和管理负载均衡器需要相应的IAM权限。（登陆时选择的是根用户则无需考虑）
2. 准备好您的EC2实例：确保它们已经正确配置，并且可以通过安全组访问。

**第二步：创建负载均衡器**

1. 登录到[AWS管理控制台](https://aws.amazon.com/console/)。
2. 在服务菜单中找到并选择“EC2”，然后在左侧导航栏中点击“负载均衡”下的“负载均衡器”。
3. 点击页面顶部的“创建负载均衡器”按钮。
4. 选择“应用型负载均衡器”（Application Load Balancer）。

**第三步：配置负载均衡器**

- 名称：为您的负载均衡器指定一个名称。
- 方案：选择互联网面向（Internet-facing），如果您的应用程序需要从互联网访问的话。
- IP版本：根据您的需求选择IPv4或双栈模式（同时支持IPv4和IPv6）。（注意这里选择的公有ipv4地址会收费，需要想方法解决，待更新） [新 – AWS 公有 IPv4 地址收费 + Public IP Insights | 亚马逊AWS官方博客](https://aws.amazon.com/cn/blogs/china/new-aws-public-ipv4-address-charge-public-ip-insights/?nc1=h_ls) 。配置ssl证书可以选择另一种方式，直接再实例上进行配置，不需要使用其它的可能产生费用的管理工具。 #待办
- VPC：选择包含您EC2实例的VPC。
- 映射：选择一个或多个可用区，并为每个选择的可用区指定一个子网。（选前两个即可）

**第四步：配置安全设置**

- 如果您计划使用之前申请好的SSL证书，请确保它已经被上传到AWS Certificate Manager (ACM) 或 IAM 中。这里可以选择“稍后添加”。（也可在这一步直接配置）

**第五步：配置安全组**

- 创建或选择一个现有的安全组，该安全组应该允许来自客户端的入站HTTPS请求（通常是端口443）。

**第六步：配置路由**

- 创建一个新的目标组或者选择已有的目标组。目标组定义了如何将请求转发给您的EC2实例。
- 指定协议（HTTP/HTTPS）、端口以及健康检查路径等信息。

**第七步：注册目标**

- 在目标组中注册您的EC2实例作为后端服务器。这使负载均衡器知道将请求发送到哪里。
- 如果之前未创建过目标组，请根据下面的步骤进行创建：

​	1. 创建新目标组

​	在目标组页面中，点击页面上方的“创建目标组”按钮。（如果是在配置过程中，可以在对应界面选择创建目标组）

​	目标类型：选择“实例”，如果您打算将请求路由到EC2实例。

​	协议和端口：对于处理HTTPS流量的应用程序，您可以选择“HTTP”或“HTTPS”。通常情况下，对于后端通信，使用HTTP即可，因为	SSL终止在负载均衡器。指定适当的端口号（例如80或自定义端口）（一般直接默认80即可）。

​	VPC：从下拉菜单中选择包含您的EC2实例的VPC。

​	名称：为您的目标组指定一个唯一且易于识别的名字。

​	健康检查设置：根据您的应用特性配置健康检查参数。默认设置通常适用于大多数情况，但您可能需要调整路径、协议等以匹配	    	您的应用程序需求。

​	2. 注册目标

​	在创建目标组的过程中，您会被提示是否立即注册目标。选择“是”，然后从可用实例列表中选择您想要添加到目标组中的EC2实例。

​	确保所选实例正在运行，并且它们的安全组允许来自负载均衡器的流量。

​	3. 完成创建

​	确认所有配置无误后，点击“创建”按钮完成目标组的创建过程。。

**第八步：完成创建**

- 审查您的设置，然后点击“创建”。

**第九步：配置监听器和SSL证书**

- 回到负载均衡器列表，选择您刚刚创建的负载均衡器，进入“监听器”选项卡。
- 添加一个HTTPS监听器，选择您在ACM中导入或申请的SSL证书。
- 配置转发规则，将流量导向之前创建的目标组。

完成以上步骤后，您的Application Load Balancer就已经配置好了，并且可以开始处理来自客户端的HTTPS请求，这些请求会被加密并通过负载均衡器安全地转发给后端的EC2实例。记得测试整个设置是否按预期工作，并调整安全组和其他设置以满足您的具体需求。



#### wordpress配置https

[WordPress HTTPS 配置问题解决方案-CSDN博客](https://blog.csdn.net/yangshuo1281/article/details/143661259?spm=1001.2101.3001.6650.11&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-143661259-blog-140606547.235%5Ev43%5Epc_blog_bottom_relevance_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-143661259-blog-140606547.235%5Ev43%5Epc_blog_bottom_relevance_base2&utm_relevant_index=12)

(注意要更改一些没有被修改的文件地址)



#### wordpress插件推荐

Import Markdown：方便导入markdown文件而且更好的保留格式。

Query Monitor：【查询监控器】WordPress开发人员工具面板。方便查看访问链接时的请求与相应，方便找到错误，调试。

SSL 不安全内容修复器：帮助清理并修复 WordPress 站点的 HTTPS 不安全内容。

#### 总结

1. 一台服务器（自带一个ip地址）
2. 一个域名
3. 通过DNS解析将域名和ip地址（服务器）联系起来。其中会有各种各样的托管工具，比如cloudflare。这需要通过在域名购买的云厂商中配置托管的方式。
4. https配置保证安全性。主要是在服务器上配置好申请的ssl证书，使其能够被识别为安全访问。在这个过程中也会有各种各样的托管工具，但必须能够直接与服务器关联，比如购买服务器的云厂商提供的工具，比如负载均衡器，这种工具通常会提供更强大的功能，但通常也会收费。也可以直接在服务器上安装证书并验证。但无法提供一些强大有用的功能(如跨可用区的高可用性等）。
5. 初次学习时，一定要学会抓住本质，不要各种各样的工具迷惑。
6. 目前对这https这一部分，以及其中涉及的各种技术还不是很熟悉，需要进一步了解。 #待办 
