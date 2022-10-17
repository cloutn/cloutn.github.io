---
title: github pages 使用随笔
date: 2022年9月19日
tags: tool
---

# github pages 的基础流程

1. 安装 hexo。（如果已安装，可以跳过这一步）
   - 先安装 NodeJs （内置 npm 工具）和 git 工具
   - 命令行运行 $ npm install -g hexo-cli 安装 hexo
   - 安装 swig 渲染器 $ npm i hexo-renderer-swig
   
2. check out https://github.com/cloutn/cloutn.github.io.git ， 并切换分支到 master

3. 打开 source\_posts 目录，用 typora 软件编辑 md 文件

   <!-- more -->

4. 编辑完成后，在根目录下依次执行以下命令：

   - hexo clean
   - **hexo generate** （每次编辑完 MD，或者增删 MD 文件之后，都必须要重新 generate 才行）
   - hexo deploy

   注意 deploy 步骤的时候，需要先开启 v2RayN 的 http 的全局代理，然后配置 tortoiseGit 的 Network 代理为 127.0.0.1:10809

5. 等待 5 分钟左右，刷新网页 https://cloutn.github.io/ 即可看到更新结果。

6. 如果想要在本地查看结果，可以在 generate 之后，执行 hexo server，来启动本地的测试服务器，然后用 http://localhost:4000/ 网址就可以访问查看结果了。

7. 最后如果确认一切都编辑完成，记得提交  https://github.com/cloutn/cloutn.github.io.git 的 master 分支，并 push 到 github remote server。

8. 编辑常用技巧：

   - shift + 回车是小换行。
   - ctl + shift + ] 是无标号的列表
   - 大标题一般前面加两个 “##” 即可，表示二级标题
   - 上传的图片保存在  source\images 目录下。

9. 添加图片的方法(目前 typora 对 hexo 的目录结构支持不好，所以需要处理一下，比较麻烦)：

   - 先把图片拷贝到 source\images 目录下
   - 把文件拖拽到 typora 中，这时会生成一个本地的绝对路径 d:/xxxx/source/images 。
   - 修改图片路径为  /images/xxxx.png



具体详细用法可以参考官方的文档：

[Hexo 官方文档](https://hexo.io/zh-cn/docs/)

[GitHub Pages 使用入门](https://docs.github.com/cn/pages/getting-started-with-github-pages)

[将 Hexo 部署到 GitHub Pages](https://hexo.io/zh-cn/docs/github-pages.html)



