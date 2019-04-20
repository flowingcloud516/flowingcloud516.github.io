本博客基于 Hexo 构建，使用 next 主题

## 命令备忘录
### 本地启动：
```bash
hexo s --debug
```
`http://localhost:4000` 打开，可观察报错

### 部署远端：
```bash
hexo d -g
```

### 写文章：
```bash
hexo new [layout] <title>
# e.g.
hexo new post 文章名
```
[Hexo-写作](https://hexo.io/zh-cn/docs/writing)

### 清除缓存
发布失败之类的情况可以用
```bash
hexo clean
```

### 详见：
https://hexo.io/zh-cn/docs/

## 工程组织结构

### git branch
1. master 分支用于部署网页（gitpage 的要求）    
2. writing 分支用于保存工作空间