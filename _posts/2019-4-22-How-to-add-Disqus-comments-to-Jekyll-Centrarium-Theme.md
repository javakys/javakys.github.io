---
layout: post
title: Jekyll Centrarium 테마에서 Disqus 댓글 기능 추가하기
date:   2019-04-21 
author: James Kim
categories: jekyll
---

Blog 포스트에는 방문자들과의 Engagement를 위해서 댓글 기능을 사용하게 되는 데, 최근에는 Disqus가 가장 널리 사용되고 있다.

Centrarium 테마는 기본적으로 Disqus 댓글 기능이 내장되어 있고 _config.yml에 그것을 위한 옵션 필드가 코멘트 처리된 상태로 제공되고 있다.

```yml
disqus_shortname: javakys
```

disqus_shortname이 Disqus 댓글 기능 사용을 위한 옵션 필드 이름이다.

Disqus에 회원 가입을 한 후에, 댓글을 기능을 추가할 Blog의 주소에 Disqus shortname으로 별칭을 부여하고 disqus shortname을 여기에 적어주면 Jekyll이 자동으로 Disqus 댓글 기능을 추가해준다.

보통의 경우에 disqus_shortname을 Disqus 가입시에 사용한 ID로 착각하는 경우가 있는데, 그것이 아니라 Disqus 기능을 사용할 URL에 일대일로 매칭되는 유일한 이름이라고 생각하면 된다.
사용자가 여러개의 Domain에서 Disqus를 사용할 수 있기 때문에 각 Domain을 구분지을 이름이 필요한데, 이것이 disqus_shortname이다.

아래 그림은 Disqus Admin Page에서 shortname("Website Name") 과 Website URL이 어떻게 서로 매팅되어 있는지에 대한 실제 예이다.

![_config.yml](/assets/images/2019-4-22/Disqus-Admin.png)