---
layout: post
title: Jekyll Centrarium Theme에서 Google Analytics 기능 추가하기
date:   2019-04-21 
author: James Kim
categories: jekyll
---

Jekyll에 Google Analytics 기능을 추가하는 가이드는 많은 Blog Post에서 제공하고 있다. 

_config.yml에 site.google_analytics={tracking ID}를 추가하고, default.html이나 base.html 등 base template file에 script 코드를 추가하는 과정이 가이드의 내용이다.

그런데, Jekyll에서 어떤 테마를 사용하는냐에 따라서 처리해야하는 설정이 달라지는 데, 이 글에서는 Centrarium 테마를 사용한 경우에 해야하는 설정에 대해서 설명한다.

Centrarium 테마는 기본적으로 Google Analytics 사용을 위한 기능이 내장되어 있고 _config.yml에 그것을 위한 옵션 필드가 코멘트 처리된 상태로 제공되고 있다.

```yml
ga_tracking_id: "UA-xxxxx-1"
```

위의 예제와 같이 ga_tracking_id가 Centrarium 테마에서 Google Analytics를 위해서 제공하는 옵션이다. 이 옵션의 값, "UA-xxxxx-1" 대신에 Google Analytics에서 Tracking하기 위한 URL을 위해 할당받은 Tracking ID를 적어주면 된다.

