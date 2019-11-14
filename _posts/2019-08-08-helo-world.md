---
layout: default
title: 你好，世界
categories: [测试]
---
<h2>{{ page.title }}</h2>
<p>我的 $ \int u \frac{dv}{dx}\,dx=uv-\int \frac{du}{dx}v\,dx $ 第一篇文章</p>

我的$$ \int u \frac{dv}{dx}\,dx=uv-\int \frac{du}{dx}v\,dx $$第一篇文章  
AAA $$\alpha$$ AA  
AAA $\alpha$ AA  
BB $\alpha$  


$$ \int u \frac{dv}{dx}\,dx=uv-\int \frac{du}{dx}v\,dx $$

$$
\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}
$$
    <p>{{ page.date | date_to_string }}</p>
    <script src="/static/mermaid.min.js"></script>
    <script>mermaid.initialize({startOnLoad:true});</script>
    <div class="mermaid">
    graph LR;
    A[aa bb]-->B(wo);
    A-->C((我是C));
    B-->D>我是D];
    C-->D;
    D-->E{我是E};
    C-->E;
    2-->E;
    _-->E;
    </div>


