---
layout: post
title: django template 정리
---

# Template

[템플릿 | Django 문서 | Django (djangoproject.com)](https://docs.djangoproject.com/ko/4.0/topics/templates/)

Two Scoops of Django - 13장 : 템플릿의 모범적인 이용



Django Template은 DTL (Django Template Language)로 구성되어 있고, 혹은 대중적인 패키지인 JinJa2가 존재한다. 또한 직접 커스텀 템플릿 백엔드를 구성할 수 도 있다.



### Syntax

#### Variables

dictionary 형태로 `{{ key }}` 라고 작성하는 경우 value값을 리턴한다.

dict가 `{'first_name' : 'John', 'last_name': 'Doe'}`인 경우 다음과 같다.

```django
My first name is {{ first_name }}. My last name is {{ last_name }}.
My first name is John. My last name is Doe
```

또한 dictionary, attribute, list-index lookup은 `.` notation으로 처리한다.

```django
{{ my_dict.key }}
{{ my_object.attribute }}
{{ my_list.0 }}
```



#### Tag

태그는 템플릿 렌더링 과정에서 로직을 제공한다.

컨텐츠를 표현할 수 도 있고, control 로직을 가질 수 도 있으며, DB로부터 데이터를 가져올 수 도 있다.

`{% something %}` 형태로 태그를 사용한다.

```django
{% csrf_token %}
{% cycle 'odd' 'even' %}
{% if user.is_authenticated %}Hello, {{ user.username }}.{% endif %}

{# 정적 파일의 내장 템플릿 태그 라이브러리를 로드 #}
{% load flavor_tags %} 

{# base.html이 부모가 되는 템플릿이므로 해당 블록을 자식 템플릿에서 이용할 수 있게 함 #}
{# 링크와 스크립트를 태그 안에 위치시키고 필요에 따라 오버라이드 되게 한다 #}
{% block style %}

{# 정적 미디어 서버에 이용될 정적 미디어 인자 #}
{% static %}
```

자세한 동작은 [Built-in template tags and filters | Django 문서 | Django (djangoproject.com)](https://docs.djangoproject.com/ko/4.0/ref/templates/builtins/#ref-templates-builtins-tags) 여기에서 확인할 수 있다.

/* csrf token 한번 공부해보기 */



#### Filter

필터는 변수나 tag arg의 값을 변경한다.

```django
{{ django|title }}
# title은 django 변수에 저장된 문자열의 첫번째 글자를 대문자로 만들어준다.
{{ my_date|date:"Y-m-d" }}
```

자세한 동작은 [Built-in template tags and filters | Django 문서 | Django (djangoproject.com)](https://docs.djangoproject.com/ko/4.0/ref/templates/builtins/#ref-templates-builtins-tags) 여기에서 확인할 수 있다.



#### Comment

```django
{# this won't be rendered #}
{% comment %}
this tag provides multi-line comments
{% endcomment %}
```



## Two scoop

템플릿의 모범적인 이용

### 13.1 대부분의 템플릿은 templates/에 넣어 두자



```
templates/
	base.html
	dashboard.html # base.html 확장
	profiles/
		base_profiles.html # base.html 확장
		profile_detail.html # base_profiles.html 확장
		profile_form.html # base_profiles.html 확장
```



- 3중 템플릿 구조
  - 각 앱은 base_<app_name>.html 템플릿을 가지고 있고, base.html 공통 템플릿을 공유
  - 각 앱안의 템플릿은 base_<app_name>.html 템플릿을 공통으로 공유
  - base.html과 같은 레벨에 있는 모든 템플릿은 base.html을 상속해서 이용



### 13.3 템플릿 주의사항

#### 1) 템플릿상 처리하는 aggregation

템플릿상에서 처리하는 aggregation method는 자제한다.

- 자바스크립트 변수에 데이터를 저장해서 iteration을 통해 체크를 하지 말고, 자바스크립트는 그 변수에 이터레이션된 값 자체를 저장하는 정도까지만 사용할 것
- 변수 값을 다 더하기 위해 add템플릿 필터를 이용하지 말 것



dateutil 라이브러리와 장고 ORM의 aggregation 메서드를 이용하는 경우 다음과 같음

/* 장고 annotate와 aggregate 참고 */

[🚦 장고 Annotate & Aggregate (velog.io)](https://velog.io/@may_soouu/장고-Annotate-Aggregate)

```python
# vouchers/managers.py
from django.utils import timezone

from dateutil.relativedelta import relativedelta

from django.db import models

class VoucherManager(models.Manager):
    def age_breakdown(self):
        """
        연령대별 카운트 수를 저장한 딕셔너리 반환
        (뷰에서 호출 후 템플릿에 context variable로 전달)
        """
        age_brackets = []
        now = timezone.now()
        
        delta = now - relativedelta(years=18)
        count = self.model.objects.filter(birth_date__gt=delta).count()
        age_brackets.append(
        	{"title": "0-17", "count": count}
        )
        count = self.model.objects.filter(birth_date__lte=delta).count()
        age_brackets.append(
        	{"title": "18+", "count": count}
        )
        return age_brackets
```

/* manager.py : model의 objects를 customizing할 수 있음 */

/* 만약 user model이 AbstractUser를 상속하고 나머지 모델이 model.Model을 상속하는 경우 사용가능 */

```django
{# templates/vouchers/ages.html #}
{% extends "base.html" %}

{% block content %}
<table>
    <thead>
    	<tr>
            <th>Age bracket</th>
            <th>Number of Vouchers Issued</th>
        </tr>
    </thead>
    <tbody>
    	{% for age_bracket in age_brackets %}
        <tr>
        	<td>{{ age_bracket.title }}</td>
            <td>{{ age_bracket.count }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock content %}
```



#### 2) 템플릿상에서의 조건문으로 하는 필터링

DB의 레코드들은 템플릿상에서 처리하기 위해 만들어진 경우가 아님. 템플릿상에서 처리하는 경우 심각하게 느려지고, 반면 PstgreSQL이나 MySQL에서는 좀 더 필터링에 최적화된 기능을 가지고 있음.

장고의 ORM이 이런 기능을 이용하는데 효과적임. 모델 매니저로 필터링 로직을 이전하면 쉽게 속도 향상을 가질 수 있음.



```python
# vouchers/views.py
from django.views.generic import TemplateView

from .models import Voucher

class GreenfeldRoyView(TemplateView):
    template_name = "vouchers/views_conditional.html"
    
    deg get_context_data(self, **kwargs):
        context = super(GreenfeldRoyView, self).get_context_data(**kwargs)
        context["greenfelds"] = \
        	Voucher.objects.filter(name__icontains="greenfeld")
        context["roys"] = Voucher.objects.filter(name__icontains="roy")
        return context
```

/* [Base views | Django documentation | Django (djangoproject.com)](https://docs.djangoproject.com/en/4.0/ref/class-based-views/base/#django.views.generic.base.TemplateView) */

get_context_data(self, **kwargs)는 object_list와 같은 기본 객체 뿐만아니라 다른 객체들도 context에 넣어보낼 수 있다.

```django
<h2>Greenfelds Who Want Ice Cream</h2>
<ul>
{% for voucher in greenfelds %}
    <li>{{ voucher.name }}</li>
{% endfor %}
</ul>
<h2>Roy Who Want Ice Cream</h2>
<ul>
{% for voucher in roys %}
    <li>{{ voucher.name }}</li>
{% endfor %}
</ul>
```



#### 3) 템플릿상에서 복잡하게 얽힌 쿼리들

장고 템플릿에서는 로직을 구현하는 것에 대해 조심해야한다.

다음과 같은 템플릿은 로직을 생성 안하는 것 처럼 보이지만 암묵적인 쿼리를 생성한다.

```django
{# list generated via User.object.all() #}
<h1>Ice Cream Fans and their favorite flavors.</h1>
<ul>
{% for user in user_list %}
    <li>
    	{{ user.name }}:
        
        {# 암묵적인 쿼리 #}
        {{ user.flavor.title }}
        {{ user.flavor.scoops_remaining }}
    </li>
{% endfor %}
</ul>
```

위 템플릿은 추출된 사용자에 대해 flavor의 field에 대한 쿼리를 수행하고 있다.

이 때 장고 ORM의 select_related() 메서드를 이용하는 것이 좋다.

select_related, prefetch_related 메서드는 하나의 쿼리셋을 가져올 때 연관되어 있는 objects들을 미리 불러오게 하는 함수이다. Join문을 사용하므로 호출되는 SQL문이 복잡해질 수는 있어도, 이렇게 불러온 데이터들은 result_cache라는 부분에 cache되기 대문에 중복호출을 방지할 수 있다.

selected_reated는 1:1의 관계 그리고 1:N관계의 N이 사용하는 정방향 참조에서의 JOIN에 유리하게 사용된다.

[[Django\]Django select_related() 와 prefetch_related() — Leffe's tistory](https://leffept.tistory.com/312)



```django
{% comment %}
List generated via User.object.all().select_related("flavors")
이렇게 연관되어 있는 objects들을 미리 불러오게 함
{% endcomment %}
<h1>Ice Cream Fans and their favorite flavors.</h1>
<ul>
{% for user in user_list %}
    <li>
    	{{ user.name }}:
        {{ user.flavor.title }}
        {{ user.flavor.scoops_remaining }}
    </li>
{% endfor %}
</ul>
```



#### 4) 템플릿에서 생기는 CPU 부하

파일 시스템(네트워크)에서 이미지를 처리하고 저장하는 작업이 템플릿 안에 존재할 경우

이미지 프로세싱 작업을 템플릿에서 분리해 뷰나 모델, 헬퍼 메서드, 또는 Celery등을 이용한 비동기 메시지 큐 시스템으로 처리한다.Message broker는 queue공간이자 task들을 처리 및 관리하는 역할을 하는 것으로 이해하면됨. (AWS의 SQS를 예로 들어볼 수 있음)

- 비동기 메시지 큐 시스템

  - worker

    - 사용자가 안보이는 곳에서 비동기적으로 큐에 작업을 넣어놓고 사용자는 업무를 볼 수 있도록 도와주는 개념. 
    - Ex. 문의하기 메일 보내기

  - Message broker

    - queue공간이자 task들을 처리 및 관리하는 역할을 하는 것으로 이해하면됨. 

    - Ex. AWS의 SQS



#### 5) 템플릿에서 숨겨진 REST API 호출

템플릿에서 객체 메서드를 호출하면 로딩 시간이 늘어남.

REST API를 호출하는 경우도 마찬가지이며 특히 서드 파티 서비스의 지도 API의 경우 매우 느리다.

이 경우 다음과 같이 해결가능하다.

- javascript 코드
  - 페이지 내용이 다 제공된 다음 클라이언트 브라우저에서 자바스크립트로 처리
- 파이썬 코드
  - 느린 프로세스를 메시지 큐, 스레드, 멀티프로세스 등의 방법으로 처리



   

### HTML 파일 작성

예시 - pg.194

base.html을 block.super를 이용하면 세가지 방법으로 상속할 수 있다.

- 기존 블락과 커스텀 블락을 동시에 사용
  - {% block %} 내에서 block.super를 쓴 다음에 커스텀 코드를 추가
- 커스텀 블락만 사용하도록 오버라이드
  - {% blcok %} 내에서 block.super를 쓰지 않고 커스텀 코드만 추가하여 오버라이드
- 기존 블락을 그대로 상속
  - 아무것도 쓰지 않고 extends만 상단에 추가



