---
layout: post
category: example2
---

<style>
.lang-align {
  text-align: left;
  margin: 1.5rem 0;
}
.lang-option {
  display: inline-flex;
  align-items: center;
  gap: 0.4em;
  margin: 0 1rem;
  cursor: pointer;
  font-size: 1.05rem;
  font-family: system-ui, sans-serif;
}
.lang-option img {
  width: 1.2em;
  height: 1.2em;
  vertical-align: middle;
}
</style>

<div id="lang-switch" class="lang-align">
  <span class="lang-option" onclick="switchLang('en')">
    <img src="https://flagcdn.com/us.svg" alt="English flag"> English
  </span>
  <span class="lang-option" onclick="switchLang('kr')">
    <img src="https://flagcdn.com/kr.svg" alt="Korean flag"> 한국어
  </span>
  <span class="lang-option" onclick="switchLang('cn')">
    <img src="https://flagcdn.com/cn.svg" alt="Chinese flag"> 中文
  </span>
</div>

{% capture lang_en %}
<!---------------------------------------------- ENGLISH SECTION ---------------------------------------------->
## English Section

This is the English content.

### Table of contents
- [The start](#the-start)
- [The middle](#the-middle)
- [The end](#the-end)

### The start

Start paragraph.

### The middle

Middle paragraph.

### The end

End paragraph.
{% endcapture %}

{% capture lang_kr %}
<!---------------------------------------------- KOREAN SECTION ---------------------------------------------->
## 한국어 섹션

이건 한국어 콘텐츠입니다.

### 시작

시작 부분입니다.

### 중간

중간 부분입니다.

### 끝

끝입니다.
{% endcapture %}

{% capture lang_cn %}
<!---------------------------------------------- CHINESE SECTION ---------------------------------------------->
## 中文部分

这是中文内容。

### 开始

开始部分。

### 中间

中间段落。

### 结束

结束内容。
{% endcapture %}

<div class="lang-block" data-lang="en">
  {{ lang_en | markdownify }}
</div>

<div class="lang-block" data-lang="kr" style="display: none;">
  {{ lang_kr | markdownify }}
</div>

<div class="lang-block" data-lang="cn" style="display: none;">
  {{ lang_cn | markdownify }}
</div>

<script>
function switchLang(lang) {
  document.querySelectorAll('.lang-block').forEach(el => {
    el.style.display = el.getAttribute('data-lang') === lang ? 'block' : 'none';
  });
}
</script>
