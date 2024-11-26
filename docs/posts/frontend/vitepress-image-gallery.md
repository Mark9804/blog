---
title: 在 VitePress 中集成图册
description: 在 VitePress 中集成 Arco Design 的多图预览组件
tags:
  - VitePress
  - Arco Design
  - Vue
  - 自定义
---

# 在 VitePress 中加入图册功能

VitePress 的设计用途是技术文档，博客功能没有 WordPress 等 CMS 强大，对于图片也没有做过多特别处理，没有常用的放大和相册功能。

之前的放大功能我是用 [medium-zoom](https://medium-zoom.francoischalifour.com/) 实现的，因为其是为完全静态的网站设计的，只能作用于当前页面，所以采取了一些[比较脏的手段](https://github.com/mark9804/blog/blob/c89a394d946145d775559db5457b0f8da10e68f7/docs/.vitepress/theme/index.mts#L23-L44)让它在路由切换时也能正常工作。

但是随着我开始记录游记，问题就出现了。我本身是一个摄影爱好者，每次出远门后经过粗筛的照片一天下来有可能达到将近 200 张，如果写文章时每张图都单独开一段的话文章内容会显得非常割裂，尺寸占比过大的图片也很影响阅读体验。因此我需要添加一个图库功能，让主题相同的图片集中在一起方便阅读。

![今年夏天出国的时候拍的，张数个位数的天数基本是因为是那天要参加学会](https://cdn.sa.net/2024/11/09/lYRAJeTcD5bzXaF.jpg =128x)

### 效果展示

:::gallery 炼金术博物馆内部
![](https://cdn.sa.net/2024/11/08/B6tznhbCkDWgsAE.webp)
![](https://cdn.sa.net/2024/11/08/C5XZL4M3NeUkWuA.webp)
![](https://cdn.sa.net/2024/11/08/Pzk5Y7hjS3NJ1E6.webp)
![](https://cdn.sa.net/2024/11/08/rz4AOiBUWJTFxfl.webp)
![](https://cdn.sa.net/2024/11/08/neQrdAT7BJ9DpkW.webp)
:::

[这篇文章](../life/view-finder-prague-1.md) 中也大量使用了图册功能。

## 实现

### 添加图库语法

我很喜欢 VitePress 提供的[自定义容器](https://vitepress.dev/zh/guide/markdown#custom-containers)功能，因此我选择了一个相似的语法作为图库语法：

```markdown
:::gallery 图册名称
![图片描述](图片链接)
![图片描述](图片链接)
![图片描述](图片链接)
:::
```

### 捕获语法

接下来需要在 VitePress 的解析阶段捕获并处理这个语法，并生成对应的 HTML 模板。VitePress 会进一步将这个模板渲染为 Vue 组件。

首先看捕获。VitePress 使用 [markdown-it](https://github.com/markdown-it/markdown-it) 作为 Markdown 解析器，因此只需要按照 API 文档添加一个自定义规则即可。

:::code-group

```ts [.vitepress/config.ts]
export default defineConfig({
  // ...
  markdown: {
    // ...
    config: md => {
      // ...
      md.core.ruler.push("gallery", state => {
        state.tokens.forEach((token, idx) => {
          if (token?.content?.includes(":::gallery")) {
            if (state.tokens[idx].type === "fence") return; // 不解析在代码块中的图册语法
            state.tokens[idx].type = "gallery";
          }
        });
        return true;
      });
    },
  },
});
```

```ts [捕获 Token 样例]
{
  type: 'inline',
  tag: '',
  attrs: null,
  map: [ 7, 10 ],
  nesting: 0,
  level: 1,
  children: [
    {
      type: 'text',
      tag: '',
      attrs: null,
      map: null,
      nesting: 0,
      level: 0,
      children: null,
      content: ':::gallery 大猪肘子',
      markup: '',
      info: '',
      meta: null,
      block: false,
      hidden: false
    },
    {
      type: 'softbreak',
      tag: 'br',
      attrs: null,
      map: null,
      nesting: 0,
      level: 0,
      children: null,
      content: '',
      markup: '',
      info: '',
      meta: null,
      block: false,
      hidden: false
    },
    {
      type: 'image',
      tag: 'img',
      attrs: [['src', 'https://example.com/image.jpg'], ['alt', '大猪肘🥰🥰🥰香香的软软的🥰🥰🥰嘿嘿🥰🥰🥰已经除了大猪肘以外什么都不会想了呢🥰🥰🥰']],
      map: null,
      nesting: 0,
      level: 0,
      children: [],
      content: '大猪肘🥰🥰🥰香香的软软的🥰🥰🥰嘿嘿🥰🥰🥰已经除了大猪肘以外什么都不会想了呢🥰🥰🥰',
      markup: '',
      info: '',
      meta: null,
      block: false,
      hidden: false
    },
    {
      type: 'softbreak',
      tag: 'br',
      attrs: null,
      map: null,
      nesting: 0,
      level: 0,
      children: null,
      content: '',
      markup: '',
      info: '',
      meta: null,
      block: false,
      hidden: false
    },
    {
      type: 'text',
      tag: '',
      attrs: null,
      map: null,
      nesting: 0,
      level: 0,
      children: null,
      content: ':::',
      markup: '',
      info: '',
      meta: null,
      block: false,
      hidden: false
    }
  ],
  content: ':::gallery 大猪肘子\n' +
    '![大猪肘🥰🥰🥰香香的软软的🥰🥰🥰嘿嘿🥰🥰🥰已经除了大猪肘以外什么都不会想了呢🥰🥰🥰](https://example.com/image.jpg)\n' +
    ':::',
  markup: '',
  info: '',
  meta: null,
  block: true,
  hidden: false
}
```

:::

### 生成 HTML 模板

接下来就是就是设置规则触发函数。由于刚刚已经加入了 `gallery` 类型，因此只需要在渲染阶段根据类型调用处理函数即可。

:::code-group

```ts [config.ts] {20-22}
// ...
import { generateImgGallery } from "./utils/generateImgGallery";

export default defineConfig({
  // ...
  markdown: {
    // ...
    config: md => {
      // ...
      md.core.ruler.push("gallery", state => {
        state.tokens.forEach((token, idx) => {
          if (token?.content?.includes(":::gallery")) {
            if (state.tokens[idx].type === "fence") return; // 不解析在代码块中的图册语法
            state.tokens[idx].type = "gallery";
          }
        });
        return true;
      });

      md.renderer.rules.gallery = (tokens, idx) => {
        return generateImgGallery(tokens[idx]);
      };
    },
  },
});
// ...
```

```ts [./utils/generateImgGallery.ts]
import { generateImgGroups } from "./generateImgGroups";

export function generateImgGallery(galleryToken: Token) {
  if (!galleryToken.children || galleryToken.children.length === 0) return "";
  const tokens = galleryToken.children;
  const galleryName = galleryToken.children[0].content
    .replace(/:::gallery\s?/, "")
    .trim();
  const imgTokens = tokens.filter(token => token.type === "image");

  return `\n<ElyImageGallery name="${galleryName}">\n${generateImgGroups(imgTokens)}\n</ElyImageGallery>\n`;
}
```

```ts [./utils/generateImgGroups.ts]
import { generateImgComponent } from "./generateImgComponent";
import { cluster } from "radash";
import type { Token } from "../../../src/types/Token";

/**
 * 生成拼接好的图片组字符串，解决直接用变量传入 non-string Object 时解析会出错的问题
 * @param {Token[]} imgListRaw 图片信息列表
 * @returns {string} 拼接好的图片组模板字符串
 */
export function generateImgGroups(imgListRaw: Token[]) {
  const imgGroups = cluster(imgListRaw, 3);
  let result = "<a-space direction='vertical'>\n";
  for (const group of imgGroups) {
    result += `<a-space>\n`;
    for (const img of group) {
      result += `${generateImgComponent(img)}\n`;
    }
    result += `</a-space>\n`;
  }
  result += `</a-space>\n`;
  return result;
}
```

```ts [./utils/generateImgComponent.ts]
import type { Token } from "../../../src/types/Token";

export function getImgInfo(imgToken: Token) {
  const src = imgToken.attrs.find(attr => attr[0] === "src")?.[1];
  const alt = imgToken.content;
  const width = imgToken.attrs.find(attr => attr[0] === "width")?.[1];
  const height = imgToken.attrs.find(attr => attr[0] === "height")?.[1];
  return { src, alt, width, height };
}

export function generateImgComponent(imgToken: Token) {
  const { src, alt, width, height } = getImgInfo(imgToken);

  const widthProps = width ? ` width="${width}"` : "";
  const heightProps = height ? ` height="${height}"` : "";

  return `\n<ArcoImage src="${src}" alt="${alt}"${widthProps}${heightProps} show-loader />\n`;
}
```

:::

### 新建 Vue 组件

最后一步就是新建 Vue 组件，并在下一步中把它注册为全局组件。

:::code-group

```vue [ImageGallery.vue]
<script setup lang="ts">
withDefaults(
  defineProps<{
    name: string;
  }>(),
  {
    name: "",
  }
);
</script>

<template>
  <a-image-preview-group>
    <a-space>
      <slot />
    </a-space>
  </a-image-preview-group>
  <figcaption
    v-if="name.length > 0"
    class="text-sm text-gray-400 text-center mt-2"
  >
    {{ name }}
  </figcaption>
</template>
```

:::

### 注册全局组件

因为我是基于 [Arco Design](https://arco.design/vue/docs/start) 进行的主题自定义，所以我会顺带全局导入 `Tooltip` 组件需要的 css。如果完全是自己写组件则不需要这一步。我使用了 [unplugin-auto-import](https://github.com/unplugin/unplugin-auto-import) 插件来自动导入组件，因此只需要在 `enhanceApp` 函数中注册即可。如果没有使用该插件的则需要手动导入 `Space` `Image` 还有 `ImagePreviewGroup`。

:::code-group

```ts [index.ts]
// ...
import ElyImageGallery from "./components/ElysiumUI/ElyImageGallery.vue";
import "@arco-design/web-vue/es/tooltip/style/css.js";
// ...

export default {
  // ...
  enhanceApp({ app }) {
    app.component("ElyImageGallery", ElyImageGallery);
  },
};
```

:::

## 一些尚待解决的小问题

至此图库组件基本上就可以使用了。不过在实际使用过程中我发现了一个会稍微恶心到我的小问题，就是它不能接受连续换行。这个需要连续解析后文来解决，但是目前已经足够我正常使用了，所以暂时就不打算修复。

```markdown
:::gallery 像这样带连续换行的语法是识别不了的

![图片描述](图片链接)

![图片描述](图片链接)

![图片描述](图片链接)
:::

:::gallery 这样是能识别的
![图片描述](图片链接)
![图片描述](图片链接)
:::
```
