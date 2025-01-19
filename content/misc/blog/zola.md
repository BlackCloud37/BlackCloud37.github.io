---
title: "从 mdbook 迁移到 Zola"
date: 2025-01-19
draft: true
taxonomies:
  categories: ["selfhost"]
  tags: ["zola"]
---

之前一直用 [mdBook](https://github.com/rust-lang/mdBook) 写博客，不过时间久了发现 book 和 blog 的定位其实不一样，book 是以目录和页面组织，丢失了时间之类的信息，于是打算迁移到其他目标就是 blog 的框架，尝试用 [Zola](https://www.getzola.org/)。

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

Zola 的内容全部放在 `content/` 目录下，有 section 和 page 的区分，同时还支持 taxonomy，例如 `tags` 和 `categories`。

在我的理解里，section 是物理(目录)上对页面文件进行组织，提供必要的导航结构。而 taxonomy 是跨目录给页面分类，提供聚合/筛选。应该是商业性质的网站，在页面比较多时需要结合使用二者。我主要用 section 就好。

`content/` 下的每个文件夹，只要里面带了 `_index.md`，就是一个 section。里面的其他 `.md` 就是该 section 的 page。如果一个 section 里还有子目录，那么子目录就是上级 section 的 subsection。整体的层次结构大概是：

```
content/
├── _index.md                   - 主页 section
├── section1/                   
│   ├── _index.md               - 主页 section.section1
│   ├── page1.md                - section1.page1
│   └── subsection1/
│       ├── _index.md           - section1.subsection1
│       └── page2.md            - section1.subsection1.page2
```

如果某个子目录只是为了组织文件，而不希望其成为一个 subsection，可以在其 `_index.md` 里添加 `transparent = true`，这样该目录下的 page 就会出现在上一级 section 的 pages 里。

## 模板语言

在开启 `list_pages = true` 后，主页会展示所有 `content/*.md`，但不包含下级 section 的 page，为了实现这一点，可以修改一下主题的 template。

```bash
rsync themes/no-style-please/templates/index.html templates/
# 之后修改 templates/ 下的对应文件，会覆盖主题的 template
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

可以看到在没开启 paginator 时，主要的渲染逻辑就是从 `section.pages` 里获取所有 page，然后列表展示，根据之前的层次结构图示，子目录下的 page 不是当前 section 的 page，自然不会展示。

因此可以把 `ul` 部分修改成：
```html
<ul>
{% for section_path in section.subsections %}
    {% set section = get_section(path=section_path) %}
    <li>
        <a href="{{ section.permalink | safe }}">{{ section.title }}</a>
        <ul>
            {% set pages = section.pages | sort(attribute="date") | reverse %}
            {% for page in pages %}
                <li>
                    <a href="{{ page.permalink | safe }}">{% if page.date and not config.extra.no_list_date %}{{ page.date }} - {% endif %}{{ page.title }}</a>
                    <br />
                    {{ page.description }}
                </li>
            {% endfor %}
        </ul>
    </li>
{% endfor %}
</ul>
```
