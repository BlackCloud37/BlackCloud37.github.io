---
title: "从 mdbook 迁移到 Zola"
date: 2025-01-19
taxonomies:
  categories: ["selfhost"]
  tags: ["zola"]
---

之前一直用 [mdBook](https://github.com/rust-lang/mdBook) 写博客，不过时间久了发现 book 和 blog 的定位其实不一样，book 是以目录和页面组织，丢失了时间之类的信息。于是打算迁移到其他 blog 框架，尝试用 [Zola](https://www.getzola.org/)。

在这里记录我是如何使用 Zola 的，一些地方不会特别详细，假设读者已经自己看过官网的文档。

## Zola Setup

安装 [zola-cli](https://www.getzola.org/documentation/getting-started/installation/)，例如 `paru -S zola`

然后创建第一个 zola 项目，options 无所谓，反正之后可以在 `config.toml` 里改。

```bash
zola init zolablog
```

zola 有很多可用 [themes](https://www.getzola.org/themes/)，我选了 [no style, please!](https://www.getzola.org/themes/no-style-please/)

安装主题：

```bash
cd zolablog/themes
git clone https://gitlab.com/4bcx/no-style-please.git
```

为了方便快速理解这个主题的页面结构，可以先尝试在本地跑一个 [live-demo](https://atgumx.gitlab.io/no-style-please) 一样的页面：

```bash
rsync themes/no-style-please/content/* content/
rsync themes/no-style-please/config.toml config.toml
# 修改一下 config.toml 的配置，且添加一行：
# theme = "no-style-please"
zola serve
```

这时候访问 [http://localhost:1111](http://localhost:1111) 应该就能看到一个和 live-demo 一样的页面了。

随后在 `config.toml` 的 `[extra]` 里添加 `list_pages = true`，这样主页应该还会把所有 `page` 按时间逆序展示出来。

## 目录结构

所有页面的**内容**都在 `content/` 目录下，每个文件夹 + `_index.md` 就是一个 section，里面的其他 `.md` 就是该 section 的 page，section 可以嵌套。如果想让文件夹不成为 section，可以在其 `_index.md` 里添加 `transparent = true`。

templates 控制每个页面的**内容**如何被渲染成 html，例如 `index.html` 控制主页的渲染，目前这些 template 都是主题自带的，为了覆盖，可以在 `templates/` 下创建同名文件，该文件会被用于覆盖主题的 template。

## 模板语言

在开启 `list_pages = true` 后，主页会展示所有 `content/*.md`，但不包含下级 section 的 page。我希望在主页展示所有近期 post，因此需要修改并覆盖主页的 template。

```bash
rsync themes/no-style-please/templates/index.html templates/
```

主题自带的主页模板是 `index.html`，内容如下：
```html 
{% extends "base.html" %}

{% block content %}
{{ section.content | safe }}

{% if config.extra.list_pages %}
{% if paginator %}
{% set pages = paginator.pages | sort(attribute="date") | reverse %}
{% else %}
{% set pages = section.pages | sort(attribute="date") | reverse %}
{% endif %}
<ul>
{% for page in pages %}
    <li>
        <a href="{{ page.permalink | safe }}">{% if page.date and not config.extra.no_list_date %}{{ page.date }} - {% endif %}{{ page.title }}</a>
        <br />
        {{ page.description }}
    </li>
{% endfor %}
</ul>
<!-- 省略了 paginator 的实现 -->
{% endif %}
{% endblock content %}
```

可以看到在没开启 paginator 时，主要的渲染逻辑就是从 `section.pages` 里获取所有 page，然后列表展示，子目录下的 page 不是当前 section 的 page，因此不会展示，将 `list_pages` 部分修改成：

```html
{% if config.extra.list_pages %}
{% if paginator %}
{% set pages = paginator.pages | sort(attribute="date") | reverse %}
{% else %}
{% set section_paths = section.subsections %}
{% set_global pages = [] %}
{% for section_path in section_paths %}
    {% set section = get_section(path=section_path) %}
    {% set_global pages = pages | concat(with=section.pages) %}
{% endfor %}
{% set pages = pages | sort(attribute="date") | reverse | slice(end=15) %}
{% endif %}

<ul>
    <li>
        posts
    </li>
    <ul>
        {% for page in pages %}
        <li>
            {% if page.date and not config.extra.no_list_date %}
                {{ page.date }}
            {% endif %}
            <a href="{{ page.permalink | safe }}">{{ page.title }}</a>
            <br />
            {{ page.description }}
        </li>
        {% endfor %}
    </ul>
</ul>
```

## Deploy gh-pages

我的 `deploy.yml`：
```yaml
name: Deploy
on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      ZOLA_VERSION: 0.19.2
    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 0
    - name: Install Zola
      run: |
        curl -sL https://github.com/getzola/zola/releases/download/v${ZOLA_VERSION}/zola-v${ZOLA_VERSION}-x86_64-unknown-linux-gnu.tar.gz | tar xz -C /usr/local/bin
    - name: Build
      run: |
        git submodule update --init --recursive
        zola build
    - name: Deploy GitHub Pages
      run: |
        git worktree add gh-pages
        git config user.name "Deploy from CI"
        git config user.email ""
        cd gh-pages
        # Delete the ref to avoid keeping history.
        git update-ref -d refs/heads/gh-pages
        rm -rf *
        mv ../public/* .
        git add .
        git commit -m "Deploy $GITHUB_SHA to gh-pages"
        git push --force --set-upstream origin gh-pages

```
