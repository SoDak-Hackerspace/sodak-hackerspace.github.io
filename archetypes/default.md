<<<<<<< HEAD
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
comments: false
images:
---

=======
+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = {{ .Date }}
draft = true
+++
>>>>>>> 57db2f765a4f59db7b6395f58cd725fb021aed4f
