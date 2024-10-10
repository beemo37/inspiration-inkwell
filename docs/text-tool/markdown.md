<!-- omit from toc -->
# Markdown参考手册

- [实现警示（Admonitions）效果](#实现警示admonitions效果)

# 实现警示（Admonitions）效果

Admonitions用于标记文档中的特别信息，如提示、警告、重要信息等。Asciidoc中内置一些警示，如`INFO`、`TIP`等，但是Markdown却不支持。

可以使用HTML + CSS的方式来构建一个Admonition，示例以及效果如下所示：

```markdown
<div style="background-color: #FFFFCC; padding: 10px;">
  <strong>💡 注意:</strong> 这是一个提示信息。
</div>
```

<div style="background-color: #FFFFCC; padding: 10px;">
  <strong>💡 注意:</strong> 这是一个提示信息。
</div>
