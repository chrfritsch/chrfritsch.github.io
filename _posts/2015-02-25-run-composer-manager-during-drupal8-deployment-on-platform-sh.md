---
layout:     post
title:      Run Composer Manager during Drupal 8 deployment on platform.sh
date:       2015-02-25
tags:       [composer]
---

If you have a module which contains a composer.json, you want to install the dependencies during the build process.


In platform.sh you have something called hooks, which are executed during the deployment.
To execute a 'drupal composer-update' during deployment, you have to add the following to your .platform.app.yaml:

{% highlight yaml %}

hooks:
 build: |
   cd public
   php modules/contrib/composer_manager/scripts/init.sh
   composer drupal-update
{% endhighlight %}
 
Now all the dependencies from all the composer.json files you have, will be downloaded after platform.sh has
built your makefile.

