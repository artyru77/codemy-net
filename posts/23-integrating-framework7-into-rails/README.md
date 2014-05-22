In this episode we show you how to setup Framework7 into your rails application.

Here is a list of code snippets that we used to setup Framework7 in our example.

`app/views/layouts/application.html+phone.erb`

```html
<!DOCTYPE html>
<html>
  <head>
    <!-- Required meta tags-->
    <%= favicon_link_tag 'favicon.ico' %>
    <%= favicon_link_tag 'apple-touch-icon-precomposed.png', rel: 'apple-touch-icon', type: 'image/png' %>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no, minimal-ui">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black">
    <!-- Your app title -->
    <title>Blog Me Now</title>
    <!-- Path to Framework7 Library CSS-->
    <%= stylesheet_link_tag    "phone", media: "all", "data-turbolinks-track" => true %>
  </head>
  <body class='phone'>
    <!-- Status bar overlay for full screen mode (PhoneGap) -->
    <div class="statusbar-overlay"></div>
    <!-- Panels overlay-->
    <div class="panel-overlay"></div>
    <%= render 'application/phone/left_panel' %>
    <!-- Views -->
    <div class="views">
      <div class="view view-main">
        <%= yield %>
      </div>
    </div>
    <%= javascript_include_tag "phone" %>
  </body>
</html>
```

`app/views/application/phone/_left_panel.html.erb`

```html
<!-- Left panel with reveal effect-->
<div class="panel panel-left panel-reveal">
  
</div>
```

`app/views/posts/index.html+phone.erb`

```html
<div class="navbar">
  <div class="navbar-inner">
    <div class='left'>
      <a href='#' class='link icon-only open-panel'>
        <i class='icon icon-bars-blue'></i>
      </a>
    </div>
    <div class='center'>
      All posts
    </div>
  </div>
</div>


<div class="pages navbar-through toolbar-through">
  <div class='page'>

  </div>
</div>
```

`app/assets/javascripts/phone/setup.coffee`

```coffee
window.F7H =
  app: new Framework7()
  dom: Framework7.$

window.Phone =
  Views: {}

Phone.Views.Main =
  F7H.app.addView '.view-main',
    dynamicNavbar: true
```