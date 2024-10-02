<p>RSSHub 是一个开源、灵活的 RSS 生成工具，允许用户为几乎任何网站创建 RSS 订阅源，即使该网站本身不支持 RSS。通过结合多种路由和参数，用户可以定制特定内容的 RSS 源，从新闻、博客到社交媒体、商品更新等。RSSHub 支持许多主流网站，如 YouTube、微博、GitHub 等，使用简单的 URL 格式即可订阅各种类型的内容。</p>
<p>它适合那些需要集中获取不同来源信息的人群，尤其是开发者或对信息流感兴趣的用户，可以自行部署或使用官方的公共实例。</p>
<p>如果你对自动化获取最新内容有需求，RSSHub 是一个很强大的工具，能够为信息追踪和聚合提供极大便利。</p>
<p>RSSHub：<a href="https://github.com/DIYgod/RSSHub">https://github.com/DIYgod/RSSHub</a></p>
<h3 id="自建-rsshub">自建 RSShub</h3>
<p>RSSHub 是一个可以将<strong>任何</strong>内容都可以抓取然后转换成 RSS 订阅的网站。</p>
<p><strong>万物皆可RSS</strong>, 它不仅可以订阅各种博客、论坛、新媒体，甚至社交媒体、推特等都不在话下，很强，详见<a href="https://docs.rsshub.app/zh/guide/">rsshub指南</a>。</p>
<p>该项目已经持续发展6年了，一直在持续更新， 甚至今年进行了一次重构。</p>
<p><img class="wp-image-1631 size-full" title="RSShub + Reeder5 利用 RSS 高效获取各大网站资讯内容-1" src="https://85box.com/wp-content/uploads/2024/10/20241002y4gf4-e1727874582200.png" alt="" width="100%"  /></p>
<p>项目支持私有部署，建议会折腾的自己可以部署一个。使用 <code>docker-compsoe</code> 很简单的就完成部署了，配置文件官方的文档中都有了 <a href="https://github.com/DIYgod/RSSHub/blob/master/docker-compose.yml">docker-compose.yml</a>。</p>
<p>对外提供访问的话，最好套一层 Nginx, 再用 <code>acme.sh</code> 来个证书自动化就完美了。</p>
<p><strong>docker-compose.yml </strong></p>
<p>加了个 acme.sh 申请证书，nginx 代理 rsshub</p>
<pre class="EnlighterJSRAW" data-enlighter-language="generic">version: '3.5'
services:
  acme:
    image: neilpang/acme.sh
    restart: always
    container_name: acme.sh
    command: ["daemon"]
    environment:
      # 我是Cloudflare DNS, 其他参考 https://github.com/acmesh-official/acme.sh/wiki/dnsapi
      - CF_Zone_ID=xxxx  
      - CF_Token=xxx
    volumes:
      - ./acme.sh:/acme.sh
      - ./certs:/ssl

  nginx:
    image: nginx
    network_mode: host
    container_name: nginx
    restart: always
    volumes:
      - ./certs:/etc/nginx/ssl
      - ./web-rsshub.conf:/etc/nginx/conf.d/rsshub.conf

  rsshub:
    image: diygod/rsshub
    restart: always
    container_name: rsshub
    ports:
      - '1200:1200'
    environment:
      NODE_ENV: production
      CACHE_TYPE: redis
      REDIS_URL: 'redis://redis:6379/'
      PUPPETEER_WS_ENDPOINT: 'ws://browserless:3000'  # marked
      GITHUB_ACCESS_TOKEN: 'xxxx'
      TWITTER_USERNAME: 'xxx'
      TWITTER_PASSWORD: 'xxx'
      TWITTER_AUTHENTICATION_SECRET: 'xxxx'
    depends_on:
      - redis
      - browserless  # marked
  
  browserless:  # marked
    image: browserless/chrome  # marked
    container_name: rsshub-browserless
    restart: always  # marked
    ulimits:  # marked
      core:  # marked
        hard: 0  # marked
        soft: 0  # marked
  
  redis:
    image: redis:alpine
    container_name: rsshub-redis
    restart: always
    volumes:
        - ./redis-data:/data
</pre>
<p><strong>web-rsshub.conf</strong></p>
<p>反向代理 rsshub 配置</p>
<pre class="EnlighterJSRAW" data-enlighter-language="generic">server {
    listen                       443 ssl;
    server_name                  rsshub.example.com;
    server_tokens                off;
    http2 on;

    ssl_certificate              /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key          /etc/nginx/ssl/key.pem;

    ssl_prefer_server_ciphers    on;
    ssl_ciphers                  EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;

    ssl_protocols                TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_session_cache            shared:SSL:50m;
    ssl_session_timeout          1d;
    ssl_session_tickets          on;

    ssl_trusted_certificate      /etc/nginx/ssl/ca.pem;
    ssl_stapling                 on;
    ssl_stapling_verify          on;

    access_log  /var/log/nginx/access_rsshub.log  main;
    error_log   /var/log/nginx/error_rsshub.log;

    resolver 8.8.8.8 ipv6=off valid=30s;

    location / {
        proxy_pass http://127.0.0.1:1200;
    }
}
</pre>
<p><strong>apply-cert.sh</strong></p>
<p>写个脚本申请证书，然后更新证书</p>
<pre class="EnlighterJSRAW" data-enlighter-language="generic">#!/bin/bash

echo "start install cert ..."

docker exec acme.sh --issue \
  -d "example.com" \
  -d "*.example.com" \
  --dns dns_cf \
  --keylength ec-256 \
  --server letsencrypt \
  --dnssleep 300 \
  --force

if [ $? -ne 0 ]; then
  echo "apply cert failed"
  exit 1
fi

docker exec acme.sh --install-cert \
  -d "example.com" \
  -d "example.com" \
  --dns dns_cf \
  --keylength ec-256 \
  --server letsencrypt \
  --key-file       /ssl/key.pem \
  --fullchain-file /ssl/fullchain.pem \
  --ca-file /ssl/ca.pem \
  --reloadcmd  "echo 'done'"

if [ $? -ne 0 ]; then
  echo "install cert failed"
  exit 1
fi

docker restart  nginx

if [ $? -ne 0 ]; then
  echo "reload nginx failed"
  exit 1
fi

echo "update cert success"
</pre>
<p>&nbsp;</p>
<p>除此之外，官方还提供了 <a href="https://docs.rsshub.app/zh/guide/#radar">Radar 功能</a>，结合浏览器插件就可以发现你正在访问的站点 RSSHub 是否已经支持订阅了，如果支持了可以一键转换成订阅的地址, 很方便。 不仅如此，还支持移动端哦。</p>
<p><img class="wp-image-1632 size-full" title="RSShub + Reeder5 利用 RSS 高效获取各大网站资讯内容-2" src="https://85box.com/wp-content/uploads/2024/10/20241002wdurh-e1727874612728.png" alt="" width="100%" /></p>
<p>当然，如果你要订阅 Github Trending, 或者 Twitter 时间线等，你需要配置一下对应的 Token, 详见<a href="https://docs.rsshub.app/deploy/config#github">配置</a>。</p>
<h3 id="reeder-5">Reeder 5</h3>
<p>RSS 的源有了，接下来就是客户端的选择了。我选择了 Reeder 5, 一个 macOS/iOS/iPadOS 上的 RSS 阅读器。 虽然收费，但是几周的体验下来，感觉还是很不错的。</p>
<p>Reeder 5 支持多种 RSS 源，包括 Feedly, Inoreader 等等，当然也支持自定义 RSS 源。</p>
<p>目前我主要用的就是订阅一些博客、公众号、推特等, 它的排版简洁，字体也还不错，而且支持 iCloud 同步，很方便。</p>
<p><img class="wp-image-1633 size-full" title="RSShub + Reeder5 利用 RSS 高效获取各大网站资讯内容-3" src="https://85box.com/wp-content/uploads/2024/10/20241002h79rq-e1727874645374.png" alt="" width="100%"  /></p>
<p>初此之外还有一个功能，稍后再读，我也很喜欢，有时候看到一些文章，当时没时间细看，可以浏览器中直接选择 <code>在 Reeder 中稍后再读</code>，有时间在打开APP慢慢看，很方便（以前都是发送到微信聊天里面…）。</p>
<p><img class="wp-image-1635 size-full" title="RSShub + Reeder5 利用 RSS 高效获取各大网站资讯内容-5" src="https://85box.com/wp-content/uploads/2024/10/202410020l6ho-e1727874696366.png" alt="" width="100%" /></p>
<p>目前这套组合体验下来，感觉不错，一些我想关注的信息，都可以通过 RSSHub 聚合到 Reeder 5 中，然后在合适的时间看，不用担心错过了。</p>
<p>然而，与此同时，信息量也变大了，自己的接受量却很有限，需要做一定的取舍，保留那些适合自己的就好了，不然反而会变成信息焦虑。</p>
<p>原文地址： <span id="sample-permalink"><a href="https://85box.com/github/rsshub.html">https://85box.com/github/<span id="editable-post-name">rsshub</span>.html</a></span> ‎</p>
