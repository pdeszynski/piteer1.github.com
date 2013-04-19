Title: Backbone.js and Zend Framework 2 Facebook like wall
Tags: Backbone.js, Zend Framework 2
Date: 2013-04-07 20:00
Category: JavaScript

# I love spaghetti code!

Yup, I really love it, cause I see it everyday looking into my old code or my teammates. The worst looking is always JavaScript code. Why? I would say it's because the JavaScript is the most powerful language. It allows you to express yourself in many ways and do almost everything. Because of that it's really important to find some good conventions.

For long time MVC pattern is used why not to use it also in JavaScript? It will help structuring the code. So we can start writing our MVC framework... but wait! There are already plenty of these!

The 3 most known MVC frameworks right now are (to be honest not exactly MVC but really close):

 * [AngularJS](http://angularjs.org/)
 * [Backbone.js](http://backbonejs.org/)
 * [Ember.js](http://emberjs.com/)

From these three I've chosen **Backbone.js** from simple reason. It allows to use existing DOM. Sadly the other ones are not meant for that. If your app would be an internal app or a SEO is not your priority at all then I would suggest using different framework than **Backbone**. But if SEO matters for you, then I would say that the best pick will be Backbone (until you would like to use for e.g. a **Phantom.js** to crawl your site and generate HTML files for **Google**).

## Foreword
This application is an 'not finished', but working example. It would need some refactoring before sending it to 'production'. I had a limited time while playing with Backbone and that's all what I was able to do. If you're interested in this tutorial, please leave a comment and I will try to refactor this code.

## Let's start with an app!
As an example app I've decided to implement simple **FB** like wall with likes mechanism. I didn't want to generate everything from JavaScript but preload data earlier. Server side part I've decided to write in PHP using **Zend Framework 2** (as a big fan of **ZF1**).

One of the important things was not writing any views twice. It's really often when you use JavaScript and PHP to write some views twice, once inside PHP as a returned response, secondary inside JS itself when some additional data is loaded through AJAX. It's really bad if you will catch yourself doing that. It leads to a lot of code duplication and need of change in many files when altering HTML structure. This can cause a lot of hard to find bugs, when pre-loaded page will work ok, but all dynamically loaded elements will cause you troubles. Let's say no to that!

### ZF2 Skeleton App
As many **ZF2** apps it will start with getting ZF2 Skeleton app from github. So firstly create a ```backbonejs``` folder inside your webserver root and then create the app using composer.phar

    :::bash
    curl -s https://getcomposer.org/installer | php --
    php composer.phar create-project --repository-url="http://packages.zendframework.com" zendframework/skeleton-application backbonejs

Alternatively (like me) instead of the root folder inside webserver you can use your workspace and [Vagrant](http://www.vagrantup.com/). I live it to your choice.

It's also necessary to setup webserver correctly to run **ZF2**, for that please refer to [instruction](http://framework.zend.com/manual/2.1/en/user-guide/overview.html) available on ZF2 documentation page.

I'm assuming that you were able to start the ZF2 application correctly and it's up and running.

Normally before you would be able to share something, you should firstly login. Because there's not much of work, let's make a login page. Not to reinvent the wheel and write it form the beginning we will use [ZfcUser](https://github.com/ZF-Commons/ZfcUser). To do it, just put inside ```composer.json``` requirement for **ZfcUser** module

    :::json
    {
        "some additional": "keys",
        "require": {
            "php": ">=5.3.3",
            "zendframework/zsendframework": "2.*",
            
            "zf-commons/zfc-user": "dev-master"
        }
    }

and run php composer.phar install. It should install you ZfcUser module.
Now let's enable this module. To do it, edit main configuration file for **ZF2** and make the **modules** array look like

    :::php
    <?php
    ///config/application.config.php
    return array(
        // This should be an array of module namespaces used in the application.
        'modules' => array(
            'Application',
            'ZfcBase',
            'ZfcUser',
        ),
        //other zf2 app settings
    );

**ZfcUser** also requires module configuration. You should copy base configuration file ```/vendor/zf-commons/zfc-user/config/zfc-user.global.php.dist```, put it inside ```/config/autoload/``` and rename it to ```zfcuser.global.php```. For the sake of this project we will enable also registration, because I would like to register myself nicely instead of writing into database my data. For that, just uncomment line inside ```zfcuser.global.php```

    :::php
    <?php
    //...
        'enable_registration' => true,
    //...
    ?>

Last step will be adding user schema into database to store somewhere bunch of users which will register. Create a database called ```social``` and run queries on it from a ```schema.mysql.sql``` file inside ```vendor/zf-commons/zfc-user/data/``` folder.

Don't forget also to setup **ZF2** to use correct database. It'll be done inside ```/config/autoload/local.php```. This file should not get inside the repository, not to store the credentials to database. Update this file to look similar to:

    :::php
    <?php
    return array(
        'db' => array(
            'username' => '<username>',
            'password' => '<password>',
        ),
    );
    ?>


You should have now working login page. Try to navigate into ```http://<server_name>/user/login```. You should see login form now. Wasn't that hard to make the login/register functionality.

But wait, it's not working yet as it should. It's fine with the login/registration but still I'm able to navigate to any page without a login in first. We will implement the functionality to disallow navigation through all application without firstly logging in. Inside the ```/modules/Application/Module.php``` (for simplicity I will use this module which came from Skeleton App) in bootstrap function update the code so it will look as follows

    :::php
    <?php
    //...
    class Module 
    {
        public function onBootstrap(MvcEvent $e)
        {
            $e->getApplication()->getServiceManager()->get('translator');
            $eventManager        = $e->getApplication()->getEventManager();
            $moduleRouteListener = new ModuleRouteListener();
            $moduleRouteListener->attach($eventManager);
            $eventManager = $e->getApplication()->getEventManager();
            //nothing's available for non logged user, so redirect him to login page
            $eventManager->attach("dispatch", function($e) {
                $sm = $e->getApplication()->getServiceManager();
                $controller = $e->getTarget();
                $auth = $sm->get('zfcuser_auth_service');
                if (!$auth->hasIdentity() && $e->getRouteMatch()->getMatchedRouteName() !== 'zfcuser/login') {
                    $application = $e->getTarget();
                    
                    $e->stopPropagation();
                    $response = $e->getResponse();
                    $response->setStatusCode(302);
                    $response->getHeaders()->addHeaderLine('Location', $e->getRouter()->assemble(array(), array('name' => 'zfcuser/login')));
                    //returning response will cause zf2 to stop further dispatch loop
                    return $response;
                }
            }, 100);
        }
        //...
    }
    ?>

Now every time you will try to go to any page other than login you will be immediately redirected to login page. And we have already a simple login implemented! Now try to login and enter the site.

## I see a wall?!
Yes, but it's... empty (or rather you're seeing a ZF2 logo). Let's cleanup it a little. ZF2 already comes with [Twitter Bootstrap](http://twitter.github.io/bootstrap/), so let's us it. We will only alter layout a little. Please open ```module/Application/view/layout.phtml``` file (yeah we'll be altering already existing Skeleton App).

    :::html
    <?php echo $this->doctype(); ?>
    <html lang="en">
        <head>
            <meta charset="utf-8">
            <?php echo $this->headTitle('MY Social Network '. $this->translate('Skeleton Application'))->setSeparator(' - ')->setAutoEscape(false) ?>

            <?php echo $this->headMeta()->appendName('viewport', 'width=device-width, initial-scale=1.0') ?>

            <!-- Le styles -->
            <?php echo $this->headLink(array('rel' => 'shortcut icon', 'type' => 'image/vnd.microsoft.icon', 'href' => $this->basePath() . '/img/favicon.ico'))
                            ->prependStylesheet($this->basePath() . '/css/bootstrap-responsive.min.css')
                            ->prependStylesheet($this->basePath() . '/css/style.css')
                            ->prependStylesheet($this->basePath() . '/css/bootstrap.css') ?>

        </head>
        <body>
            <div class="navbar navbar-inverse navbar-fixed-top">
                <div class="navbar-inner">
                    <div class="container">
                        <a class="btn btn-navbar" data-toggle="collapse" data-target=".nav-collapse">
                            <span class="icon-bar"></span>
                            <span class="icon-bar"></span>
                            <span class="icon-bar"></span>
                        </a>
                        <a class="brand" href="<?php echo $this->url('home') ?>"><?php echo $this->translate('MY Social Network') ?></a>
                        <div class="nav-collapse collapse">
                            <ul class="nav">
                                <?php if ($this->zfcUserIdentity()): ?>
                                    <li class="active"><a href="<?php echo $this->url('home') ?>"><?php echo $this->translate('Home') ?></a></li>
                                    <li class="active"><a href="<?php echo $this->url('zfcuser') ?>"><?php echo $this->translate('Me') ?></a></li>
                                    <li class="active"><a href="<?php echo $this->url('zfcuser/logout') ?>"><?php echo $this->translate('Logout') ?></a></li>
                                <?php else: ?>
                                    <li class="active"><a href="<?php echo $this->url('zfcuser/login') ?>"><?php echo $this->translate('Login') ?></a></li>
                                <?php endif; ?>
                            </ul>
                        </div><!--/.nav-collapse -->
                    </div>
                </div>
            </div>
            <div class="container" id="container">
                <?php echo $this->content; ?>
                <hr>
                <footer>
                    <p>&copy; 2013 - <?php echo date('Y') ?> My super company <?php echo $this->translate('All rights reserved.') ?></p>
                </footer>
            </div> <!-- /container -->
            <!-- Scripts -->
            
            <?php echo $this->headScript()
                            ->prependFile($this->basePath() . '/js/backbone.js')
                            ->prependFile($this->basePath() . '/js/underscore.js')
                            ->prependFile($this->basePath() . '/js/html5.js', 'text/javascript', array('conditional' => 'lt IE 9',))
                            ->prependFile($this->basePath() . '/js/bootstrap.min.js')
                            ->prependFile($this->basePath() . '/js/jquery.min.js'); ?>

            <?php echo $this->inlineScript() ?>
        </body>
    </html>

Here are some important notes. You saw where the scripts went? Almost at the end of body.
    
    :::html
     <!-- Scripts -->
    <?php echo $this->headScript()
                    ->prependFile($this->basePath() . '/js/backbone.js')
                    ->prependFile($this->basePath() . '/js/underscore.js')
                    ->prependFile($this->basePath() . '/js/html5.js', 'text/javascript', array('conditional' => 'lt IE 9',))
                    ->prependFile($this->basePath() . '/js/bootstrap.min.js')
                    ->prependFile($this->basePath() . '/js/jquery.min.js'); ?>

    <?php echo $this->inlineScript() ?>

Why you ask? When your social network will be growing you will start adding more and more JavaScript. There are many methods of optimization, but one of the simplest one will be moving your scripts at the end of HTML. This will make browser parse the scripts after showing the rest of the page to an user. There's one minus though - until scripts are loaded user cannot interact with your app. This problem can be solved but I won't be doing it here.
It's important to have them added in the same order. First thing that should be loaded is jQuery. This is one of the dependencies required for **Backbone.js**. Other one is **Underscore.js**. This JavaScript has to be also inside the DOM before **Backbone** (we use here ```prependFile``` function). Last included JavaScript is **Backbone.js** itself.

    :::html
    <ul class="nav">
    <?php if ($this->zfcUserIdentity()): ?>
        <li class="active"><a href="<?php echo $this->url('home') ?>"><?php echo $this->translate('Home') ?></a></li>
        <li class="active"><a href="<?php echo $this->url('zfcuser') ?>"><?php echo $this->translate('Me') ?></a></li>
        <li class="active"><a href="<?php echo $this->url('zfcuser/logout') ?>"><?php echo $this->translate('Logout') ?></a></li>
    <?php else: ?>
        <li class="active"><a href="<?php echo $this->url('zfcuser/login') ?>"><?php echo $this->translate('Login') ?></a></li>
    <?php endif; ?>
    </ul>

ZfcUser gives you out of the box functionality to check whether user is logged or not. You can do it by calling

    :::php
    <?php
    $this->zfcUserIdentity();
    ?>
in a view file. Right now our social network is really simple and allows only to show wall or user info (this is also provided by **ZfcUser**) and eventually log out.

Let's also add a little bit of CSS styles for later usage (they're really simple not to make the first steps too complex, style the application as you would like).

    :::css
    /* public/css/style.css */
    .border-bottom {
      border-bottom:1px solid #ccc;
      margin-bottom: 5px;
    }

    .border-left {
      border-left:1px solid #ccc;
    }

    .border-right {
      border-right:1px solid #ccc;
    }

    div.share-form {
      padding:5px;
    }

    textarea#share-input {
      height: 40px;
      width: 100%;
      -moz-border-radius: 5px;
      margin: 0; padding: 0;
      border-radius: 5px;
    }

    div.post {
      padding: 10px;
    }

    div.post-footer {
      color: #3b5998;
      font-size: 12px;
    }

    p.post-author {
      color: #3b5998;
    }

    .btn-share {
      float:right;
      margin: 5px 0 5px 0;
    }

    span.post-like-likes {
      position: absolute;
      padding: 4px;
      background-color: #000;
      color: #fff;
      -moz-box-shadow: 5px 5px 2px #aaa;
      -webkit-box-shadow: 5px 5px 2px #aaa;
      box-shadow: 5px 5px 2px #aaa;
      -moz-border-radius: 5px;
      border-radius: 2px;
      display:none;
    }

Ok, so we can assume that layout is done. Now you should see simple menu with 3 tabs on the top, but still no posts.

### My share form
What kind of social network would be without a share form? We need one. Let's do it then.

    :::php
    <?php
    //module/Application/src/Application/Form/ShareForm.php
    <?php
    namespace Application\Form;

    use Zend\Form\Form;

    class ShareForm extends Form
    {
        public function __construct()
        {
            parent::__construct('share');
            $this->add(
                array(
                    'name' => 'share',
                    'options' => array(
                        'label' => "Status",
                    ),
                    'attributes' => array(
                        'type'  => 'textarea',
                        'id'    => 'share-input',
                    ),
                )
            );

            $this->add(
                array(
                    'name' => 'submit',
                    'attributes' => array(
                        'type' => 'submit',
                        'value' => 'Share',
                        'class' => 'btn btn-primary btn-share',
                    ),
                )
            );
        }
    }
This will create a simple form with a textarea and a submit button. We will use later **id** attribute to attach events to this form. Also not that the form will be without any **Filter** nor validation. In a production application validation should be added (ZF2 Forms support it really nicely).

Next step will be putting this form inside view for our wall (till now only layout exists). Create a view inside of ```module/Application/src/view/application/index``` and name it ```index.phtml```.

    :::php
    <div class="container">
        <div class="row">
            <div class="span1">
                <p>sidebar left</p>
            </div>
            <div class="content span6 border-left">
                <div class="border-bottom share-form">
                    <?php 
                    $this->form->prepare();
                    $this->form->setAttribute('action', $this->url('application/default', array('controller' => 'share', 'action' => 'add')));
                    $this->form->setAttribute('id', 'share-form');
                    $this->form->setAttribute('method', 'post');

                    //open form
                    echo $this->form()->openTag($form);
                    echo $this->formRow($this->form->get('share'));
                    echo '<br />';
                    echo $this->formRow($this->form->get('submit'));
                    echo $this->form()->closeTag();
                    ?>
                    <br clear:both; />
                </div>
            </div>
            <div class="span2">
                <p>sidebar right</p>
            </div>
        </div>
    </div>
In the code above you will see a 3 column layout for the wall page (later you will want probably to put there some additional navbar). For now left and right sidebars will be empty.

    :::php
    <?php
     $this->form->setAttribute('action', $this->url('application/default', array('controller' => 'share', 'action' => 'add')));
     ?>
Also already an url for a form was added (for now not working) where your new shares will be put. This will generate a form with textarea and a button under it, but the form is still not inside the view. Later we will pass it to the view from controller with all shares. If you enter the index page there will be and error so wait some more time.

### Models to get'em all
To obtain shares of users model will be necessary. For simplicity there will be only one table, and you will be able to see for now (sadly) only your shares. Create schema inside your **social** database.

    :::sql
    CREATE TABLE `share` (
      `share_id` int(11) unsigned NOT NULL AUTO_INCREMENT,
      `user_id` int(11) unsigned NOT NULL COMMENT 'User id of an the posts should be visible',
      `author_id` int(11) unsigned NOT NULL,
      `author` varchar(45) NOT NULL COMMENT 'Name of author of post',
      `text` text NOT NULL COMMENT 'Share text',
      `type` tinyint(5) NOT NULL DEFAULT '0' COMMENT 'Has link',
      `thumbnail` varchar(255) DEFAULT NULL COMMENT 'Optional thumbnail',
      `added` datetime NOT NULL,
      `likes` int(10) unsigned NOT NULL DEFAULT '0',
      PRIMARY KEY (`share_id`),
      KEY `user_idx` (`user_id`),
      KEY `author_idx` (`author_id`),
      KEY `user_id_added` (`user_id`,`added`),
      CONSTRAINT `author` FOREIGN KEY (`author_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE NO ACTION,
      CONSTRAINT `user` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE NO ACTION
    ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8

With each share there will be stored likes counter. It is necessary to already show how many likes one share will have without using counts for each post. Now let's create a **ShareTable** which will be a **TableGateway**

    :::php
    <?php
    namespace Application\Model;

    use Zend\Db\TableGateway\TableGateway;
    use Zend\Db\Sql;

    class ShareTable extends TableGateway
    {
        protected $tableGateway;

        public function __construct(TableGateway $tableGateway)
        {
            $this->tableGateway = $tableGateway;
        }

        public function getByUserId($userId)
        {
            return $this->tableGateway->select(function(Sql\Select $select) use ($userId) {
                $select
                    ->order('added DESC')
                    ->limit(20)
                    ->where
                        ->equalTo('user_id', $userId);
            });
        }

        public function insert($data)
        {
            $this->tableGateway->insert(array(
                'user_id' => $data['user_id'],
                'author_id' => $data['author_id'],
                'author' => $data['author'],
                'text' => $data['text'],
                'type' => $data['type'],
                'added' => new Sql\Expression('NOW()'),
            ));

            return $this->tableGateway->lastInsertValue;
        }

        public function updateLikes($shareId, $amount = 1)
        {
            $this->tableGateway->update(array(
                    'likes' => new \Zend\Db\Sql\Expression('likes + ' . (int)$amount)
                ),
                array('share_id' => $shareId)
            );
        }
    }
Hope that the code is self explanatory this time.

Because the same model can be used later in different places inside our module, the best way will be registering it as a factory. This will allow you to use the same instance of ShareTable across the system. Let's register it as a factory, cause it's dependent on TableGateway, which is dependent on ```Zend\Db\Adapter\Adapter```. Open ```module/Application/Module.php``` and make ```getServiceConfig()``` function to look as follows

    :::php
    <?php
    //module/Application/Module.php

    namespace Application;

    use Zend\Mvc\ModuleRouteListener;
    use Zend\Mvc\MvcEvent;
    use Zend\Db\TableGateway\TableGateway;

    use Application\Model;
    use Application\View;

    class Module
    {
        //...
        public function getServiceConfig()
        {
            return array(
                'factories' => array(
                    'Application\Model\ShareTable' => function ($sm) {
                        $tableGateway = $sm->get('ShareTableGateway');
                        $table = new Model\ShareTable($tableGateway);
                        return $table;
                    },
                    'ShareTableGateway' => function ($sm) {
                        $dbAdapter = $sm->get('Zend\Db\Adapter\Adapter');
                        
                        return new TableGateway('share', $dbAdapter);
                    },
                )
            );
        }
    }
So we have now 2 services ```Application\Model\ShareTable``` and ```ShareTableGateway```. Also a nice idea would be registering prototype row for ```ShareTableGateway``` but we'll omit that for now.

### Controller for gluing all the pieces together
Now it's time for our controller to glue everything and start viewing some shares (if there are any).

    :::php
    <?php
    //module/Application/src/Application/Controller/IndexController.php

    namespace Application\Controller;

    use Zend\Mvc\Controller\AbstractActionController;
    use Zend\View\Model\ViewModel;

    use Application\Form;
    use Application\Model;

    class IndexController extends AbstractActionController
    {

        protected $shareTable;

        public function indexAction()
        {
            $form = new Form\ShareForm();
            $shareTable = $this->getShareTable();
            $userId = $this->zfcUserAuthentication()->getIdentity()->getId();
            return new ViewModel(
                array(
                    'form' => $form,
                    'posts' => $this->getShareTable()->getByUserId($userId),
                )
            );
        }

        public function getShareTable()
        {
            if (!$this->shareTable) {
                $sm = $this->getServiceLocator();
                $this->shareTable = $sm->get('Application\Model\ShareTable');
            }

            return $this->shareTable;
        }
    }
Function ```indexAction``` will render our wall with all shares that are specified for currently logged user. Note that here we use **Inversion of Control** for our share table.

The last step will be altering our view by adding some code responsible for viewing each share. Modify the ```module/Application/view/index/index.phtml``` file so it will now look like

    :::php
    <div class="container">
        <div class="row">
            <div class="span1">
                <p>sidebar left</p>
            </div>
            <div class="content span6 border-left">
                <div class="border-bottom share-form">
                    <?php 
                    $this->form->prepare();
                    $this->form->setAttribute('action', $this->url('application/default', array('controller' => 'share', 'action' => 'add')));
                    $this->form->setAttribute('id', 'share-form');
                    $this->form->setAttribute('method', 'post');

                    //open form
                    echo $this->form()->openTag($form);
                    echo $this->formRow($this->form->get('share'));
                    echo '<br />';
                    echo $this->formRow($this->form->get('submit'));
                    echo $this->form()->closeTag();
                    ?>
                    <br clear:both; />
                </div>
                <div id="post-list">
                    <?php foreach ($this->posts as $post): ?>
                        <?php echo $this->partial('index/partials/post.phtml', array('post' => $post)); ?>
                    <?php endforeach; ?>
                </div>
            </div>
            <div class="span2">
                <p>sidebar right</p>
            </div>
        </div>
    </div>
Each post will be rendered by partial. This is because the view will be cleaner and easier to read this way. Additionally it's because of **DRY**. We will reuse this partial later for JavaScript. This partial looks like

    :::php
    <div class="post" id="post_<?php echo $this->post['share_id']?>" >
        <div class="border-bottom">
            <div>
                <p>
                    <span class="gravatar">
                        <?php if (!isset($this->post['gravatar'])): ?>
                            <?php echo $this->gravatar($this->post['author'], array('img_size' => 40)); ?>
                        <?php else: ?>
                            <?php echo $this->post['gravatar']; ?>
                        <?php endif; ?>
                    </span>
                    <span class="post-author">
                        <?php echo $this->post['author']; ?>
                    </span>
                </p>
            </div>
            <p class="post-text"><?php echo $this->post['text']; ?></p>
            <div class="post-footer">
                <span class="post-like">
                    <a class="post-like-button">Like!</a>
                    <span class="post-like-likes"></span>
                    <span class="post-likes-count"><?php if ($this->post['likes']): ?>(<?php echo $this->post['likes']; ?>)<?php endif; ?></span>
                </span>
                 &middot; 
                <span class="post-date"><?php echo $this->post['added']; ?></span>
            </div>
        </div>
    </div>
As you can see we will be showing a gravatar, author and share text

If you will add one record to shares table you should already see a new share. But still we cannot create new shares from our Social Network.

### It's time for some JavaScript action
Now let's make some action with backbone. Let's create new file ```share.js``` and put it inside ```public/javascript``` folder. Let's also add this file to included JavaScript list. It can be done by placing on the beginning of ```module/Application/view/index/index.phtml``` a line

    :::php
    <?php $this->headScript()->appendFile($this->basePath() . 'js/share.js'); ?>
This will do what we need (again for the simplicity there will be only one file, normally you would divide this app into more files and during the deployment merge them correctly or use something like Require.js). Also be sure to use this time (in opposition to layout) ```appendFile``` function to be sure that this script will be loaded **after** Backbone.js.

Let's start editing ```share.js``` file and put some JS into an action. Firstly let's create new module and main application object with namespaces for it's parts.

    :::javascript
    (function(share, global, Backbone, $, user, undefined){
        'use strict';
        //application - manages input and send messages to collection
        var AppView = Backbone.View.extend({
            el: $('#container')
        });
        AppView.models = {};
        AppView.collections = {};
        AppView.views = {};


        $(function() {
            new AppView();
        });
    })({}, this, Backbone, $, user);
Our main application object is an Backbone view! This way we'll be able to 'glue' to already existing DOM and add some events for already existing elements. To make it glued to the already existing DOM Backbone requires to put ```el: domElement``` inside the view (it's also possible to specify this element while constructing objects).

Additionally there were created 3 namespaces to organize our code a little (models, collections, views).

When you use Backbone you divide your code between **Views, Models and Collections** Model's responsible for logic and storing data. Collection is just a container of many models. View at the end is responsible for rendering HTML.

Firstly let's create a model for a single share entry.
    
    :::javascript
    AppView.models.ShareModel = Backbone.Model.extend({
    });

And a view

    :::javascript
    AppView.views.ShareView = Backbone.View.extend({
        model: AppView.models.ShareModel,
        tagName: 'div',
        template: _.template($('#post-template').html()),
        className: 'post',
        attributes: function() {
            return {
                id: 'post_' + this.model.get('share_id'),
            };
        },
        initialize: function(attr) {
            this.listenTo(this.model, 'change', this.render);
            this.listenTo(this.model, 'destroy', this.remove);

            this.render();
        },
        render: function() {
            var attrs = _.clone(this.model.attributes);
            //print readable date
            attrs.added = attrs.added.toLocaleString();
            this.$el.html(
                    $.parseHTML(this.template(attrs))[1]
                        .innerHTML
                );
            return this;
        },

        

        hideLikes: function() {
            this.likesList.remove();
        },

        remove: function() {
            this.$el.remove();
            return this;
        }
    });

There's a bunch of code, so let's stop a little.

It's really common that each view in Backbone has it's own model. By putting inside of view definition
    
    :::javascript
    //...
    model: AppView.models.ShareModel
we define that the default model for such a view is ```AppView.models.ShareModel```

Next thing is ```tagName```, this just tells backbone to use ```div``` as a main tag for element (each view has it's own 'container'). Let's skip few lines and firstly look into class constructor. In Backbone constructor is an ```initialize``` method.

    :::javascript
    //...
    initialize: function(attr) {
        this.listenTo(this.model, 'change', this.render);
        this.listenTo(this.model, 'destroy', this.remove);

        this.render();
    },
This will make us life easier. Each time there's any change inside of the model, the view will be informed. So if for e.g. Edit functionality will be implemented and model will be altered, then the view will be automatically updated.
Right now also in case when model is destroyed, view with a share will be removed from DOM. Because of the ```this.render()``` function code each time new share view will be created immediately it will go into the DOM.

Now let's analyze render and template function. Backbone.js uses by default underscore templating mechanism. You can create templates in at least two ways

 * Create template server side. This template will go inside the HTML page and will be obtained from JavaScript.
 * Create template inside your JS file with a view.

Which method to use, differs from situation. For me important is **DRY**. The worst thing is making repetitions of templates and creating two, one for server side rendering, second for dynamically created content. This way you will end altering twice your templates in case of any change. So here I've decided to reuse my partial for each share.

Somewhere at the end of ```module/Application/view/index/index.phtml``` put this code

    :::php
    <script type="text/template" id="post-template">
    <?php echo $this->partial('index/partials/post.phtml', array(
        'post' => $this->templateVars(array(
            'share_id',
            'author',
            'text',
            'added',
            'likes',
            )) + array(
                'gravatar' => '<img src="<%- gravatar %>" />',
            )
        )
    ); ?>
    </script>
    <script type="text/javascript">
    var user = {
        email: <?php echo json_encode($this->zfcUserIdentity()->getEmail()); ?>,
        id:    <?php echo json_encode($this->zfcUserIdentity()->getId()); ?>
    }
    </script>

This way a template for Backbone will be created and it fully reuses previously defined template for a single share! Also here, we've passed some info for JavaScript about the currently logged user.

If you look closely you will see ```$this->templateVars(..)``` call. This is a custom helper for helping us with Underscore templates.
Normally underscore requires templates like
    
    :::javascript
    <%- variable_name %>
Because our partial waits for model like

    :::php
    <?php
    array(
        'text' => 'here goes some text',
        //...
    );

Let's cheat it a little and pass him an array like

    :::php
    <?php
    array(
        'field' => '<%- field %>'
    );
Let's create then **TemplateVars** helper by putting class inside of the ```module/Application/src/Application/View/Helper/TemplateVars```
    
    :::php
    <?php
    //module/Application/src/Application/View/Helper/TemplateVars
    namespace Application\View\Helper;
     
    use Zend\Http\Request;
    use Zend\View\Helper\AbstractHelper;
     
    class TemplateVars extends AbstractHelper
    {
        /**
         * Changes variables array into backbone's template vars syntax
         *
         * @param array $variables Template variables names
         *
         * @return array Array of template vars
         */
        public function __invoke(array $variables)
        {
            $output = array();
            foreach ($variables as $var) {
                $output[$var] = sprintf('<%%- %s %%>', $var);
            }

            return $output;
        }
    }
Additionally it's required to register it for view, put this code inside ```module.config.php```

    :::php
    <?php
    //...
    return array(
        //....
        'view_helpers' => array(
            'invokables' => array(
                'templateVars' => 'Application\View\Helper\TemplateVars',
            ),
        ),
    );
Now we can come back to our Backbone view. Remember
    
    :::javascript
    template: _.template($('#post-template').html())
This way our view will obtain a template for each share. Having our template it's possible now to render each new share!

Here comes one of the Backbone.js flaws. It requires a lot of boilerplate. Backbone views doesn't know how to render themselves. It's necessary to create a render function. As it was shown before, render function looked like:

    :::javascript
    render: function() {
        var attrs = _.clone(this.model.attributes);
        //print readable date
        attrs.added = attrs.added.toLocaleString();
        this.$el.html(
                $.parseHTML(this.template(attrs))[1]
                    .innerHTML
            );
        return this;
    },

Rendering almost always goes this way
 * Take the model
 * Pass it to the template
 * Set is as the main element content (```this.$el.html(...)```)

Additionally two tricks were done here. One is to change the raw ```Date``` into some more readable one. It's done by calling ```toLocaleString()```. Still it would be bad to loose the raw date object because it might be necessary to check later the date of post, that's why it was done not on original model attributes but on it's copy.

Second trick is that we take innerHTML of rendered template. If it wouldn't be done here then our structure would be different than the prerenderred posts. It would look like:

    :::html
    <div>
        <div class="post" id="post_<some_id>">
            //....
        </div>
    </div>
Calling innerHTML will take only contents of ```<div class="post"></div>```

To store inside the DOM also share id of newly created share **className** and **attributes** were defined. Attributes are a function because they're dynamically created. In other way you would finish with the same id in every view created now or later (because attribute would be created JavaScript parse time).

In Backbone it's also quite common to define a list of views. This is an additional View object which handles whole list of single views. This kind of 'View Lists' are mostly bound to collections instead of model. In our case we will create **ShareViewList** view which will hold whole list of shares.

    :::javascript
    //represents view of whole wall
    AppView.views.ShareViewList = Backbone.View.extend({
        el: $('#post-list'),
        initialize: function() {
            this.listenTo(this.collection, 'add', this.add);

            //initialize already existing views in DOM
            for (var i = 0, len = this.collection.length; i < len; ++i) {
                var element = this.collection.models[i];
                new AppView.views.ShareView({model: element, el: $('#post_' + element.get('share_id'))});
            }
        },

        add: function(data) {
            var view = new AppView.views.ShareView({model: data, id: data.share_id});
            this.$el.prepend(view.el);
        }
    });
Let's stop on important things here. One thing is that we listen to Collection **add** event here. Every time something is added to a collection it will create new ShareView and this way new HTML will be created. 
Second thing is that during creating an instance of this class collection will be searched for already created shares (for loop). Thanks to it, all already created shares will have it's View inside of JS (and all events which ShareView defines attached).

Let's move now to ShareCollection which looks as follows

    :::javascript
    //represents whole wall of posts
    AppView.collections.ShareCollection = Backbone.Collection.extend({
        model: AppView.models.ShareModel,
        comparator: function(a, b) {
            return a.added > b.added;
        },
        initialize: function() {
            var list = [], self = this;
            $('#post-list > .post').each(function(index, el){
                var $likesEl = $(el).find('.post-likes-count');
                list.push(new self.model({
                    share_id: $(el).attr('id').replace(/post_/, ''),
                    author:   $(el).find('.post-author').text(),
                    text:     $(el).find('.post-text').text(),
                    added:    new Date($(el).find('.post-date').text()),
                    gravatar: get_gravatar($(el).find('.post-author').text().trim(), 40),
                    likes:    $likesEl.text() !== '' ? parseInt($likesEl.text().replace(/[^0-9]+/g, '')) : 0
                }))
            });
            this.add(list);
        }
    });
This collection when it's created will try to look inside the DOM for already created shares (which are later used inside the ShareViewList to create single views). This way you will have data that is already inside of your HTML (to be honest I think this code should go inside the ShareViewList to encapsulate it completely there, but I didn't have enough time to refactor it).

There's an information for Backbone that this collection will use ```model: AppView.models.ShareModel```. This way you know what kind of model to expect.

I've also used here get_gravatar function (put it at the beginning of our module or in separate file) found [here](http://www.deluxeblogtips.com/2010/04/get-gravatar-using-only-javascript.html).

### Some server side PHP
Ok, so basically we have some code to create new Views representing each share, but still they do not do anything. Let's create some code responsible for adding new shares.

    :::php
    <?php
    namespace Application\Controller;

    use Zend\Mvc\Controller\AbstractActionController;
    use Zend\View\Model\ViewModel;

    use Application\Form;
    use Application\Model;

    class ShareController extends AbstractActionController
    {

        protected $acceptCriteria = array(
            'Zend\View\Model\JsonModel' => array(
                'application/json',
            ),
        );

        protected $shareTable;

        public function addAction()
        {
            $viewModel = $this->acceptableViewModelSelector($this->acceptCriteria);
            $shareTable = $this->getShareTable();

            return $viewModel->setVariables(array(
                    'share' => $shareTable->insert(array(
                        'user_id' => $this->zfcUserAuthentication()->getIdentity()->getId(),
                        'author_id' => $this->zfcUserAuthentication()->getIdentity()->getId(),
                        'author' => $this->zfcUserAuthentication()->getIdentity()->getEmail(),
                        'text' => $this->params()->fromPost('text'),
                        'type' => 1,
                    )
                ))
            );
        }

        public function deleteAction()
        {
            $viewModel = $this->acceptableViewModelSelector($this->acceptCriteria);
            return $viewModel->setVariables(array(
                    'deleted' => $this->getShareTable()->delete(
                        array('share_id' => $this->params()->fromPost('share_id'))
                    )
                )
            );
        }

        public function getShareTable()
        {
            if (!$this->shareTable) {
                $sm = $this->getServiceLocator();
                $this->shareTable = $sm->get('Application\Model\ShareTable');
            }

            return $this->shareTable;
        }
    }
Important part is ```addAction``` function. It just gets data sent by post and pass them to ```ShareTable``` model.
I've also used here context which are available in *ZF2*. Why? To be able to post something either using JavaScript or normal **POST** request. Thanks to it, again, I won't have to repeat myself.

It's important also to register our new controller inside of the module.config.php file by adding new invokable.

Look for ```controllers``` key and modify it
    
    :::php
    <?php
    return array(
        'controllers' => array(
            'invokables' => array(
                'Application\Controller\Index' => 'Application\Controller\IndexController',
                'Application\Controller\Share' => 'Application\Controller\ShareController',
            ),
        ),
    );

### Events, events, events
It's time to make our share form working. To achieve it, we have to catch some user events. Let's come back to our **AppView** class. We'll extend it by adding some code inside.

    :::javascript
    //application - manages input and send messages to collection
    var AppView = Backbone.View.extend({

        el: $('#container'),
        initialize: function() {
            this.input      = $("textarea[name='share']");
            this.form       = $("form[name='share']");
            this.collection = new AppView.collections.ShareCollection();
            this.shareViewList = new AppView.views.ShareViewList({collection: this.collection});
        },
        events: {
            "keypress #share-form": "createOnEnter",
            "submit #share-form": "createOnClick",
        },

        createOnClick: function(event) {
            event.preventDefault();
            this.create();
        },
        createOnEnter: function(event) {
            //if keypress other than enter no action
            if (event.keyCode !== 13 || !event.shiftKey) return;
            //disable submit
            event.preventDefault();
            event.stopPropagation();
            this.create();
        },
        create: function() {
            var self = this;
            if (!self.input.val().trim()) return;
            //ajax action to add element
            self.input
                .attr('disabled', 'disabled')
                .blur();
            $.post(self.form.attr('action'), {
                    'text': self.input.val().trim()
                }, function(response) {
                    self.collection.add(new AppView.models.ShareModel({
                        share_id: response.share,
                        text: self.input.val().trim(),
                        likes: 0,
                        added: new Date(),
                        author: user.email,
                        gravatar: get_gravatar(user.email.trim(), 40),
                        validate: true,
                    }));
                },
                'json'
            )
            .always(function () {
                self.input
                    .attr('disabled', false)
                    .focus();
            });
        }
    });
In Backbone we define events as follows

    :::javascript
    events: {
        "keypress #share-form": "createOnEnter",
        "submit #share-form": "createOnClick",
    },
How to read it? You can translate it for yourself: *Bind keypress event to a form with id share-form which is inside this.$el and call createOnEnter function defined inside this view.* Simple!

Let's look a little more inside the create function. Here using jQuery **post** function data is sent to an URL defined as an action attribute inside share form. It's important to add after the response callback ```json``` datatype. This will make **jQuery** to set correct **Accept** header and **ZF2** will return json as an response.

Backbone allows to do it differently using the ```save()``` method. I will show how it's possible using Likes functionality. This time let's stick to the jQuery as an example of different approach.

Important part here is that we don't use explicitly View at all in *success* callback. It's enough to add a model to a collection and *View* will be created.

Now you should be able to post something using the **Share** button or by pressing **Shift+Enter** buttons.

### I like what you wrote
It would be nice to show that you liked something (sorry no dislikes). Let's implement it also. Firstly let's create schema for future likes

    :::sql
    CREATE TABLE `likes` (
      `like_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      `share_id` int(10) unsigned NOT NULL,
      `user_id` int(10) unsigned NOT NULL,
      `user_name` varchar(255) NOT NULL,
      PRIMARY KEY (`like_id`),
      UNIQUE KEY `share_id_user_id` (`share_id`,`user_id`),
      KEY `author_idx` (`user_id`),
      KEY `share_idx` (`share_id`),
      CONSTRAINT `share_idx` FOREIGN KEY (`share_id`) REFERENCES `share` (`share_id`) ON DELETE NO ACTION ON UPDATE NO ACTION,
      CONSTRAINT `user_idx` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE NO ACTION
    ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;


Now let's create model, collection, single like view, and likes list. This time we won't be using server side templates, because **Likes** won't be created at all from a server and will be loaded on demand while hovering above **Like** button.

    :::javascript
    AppView.models.LikesModel = Backbone.Model.extend({
        idAttribute: 'like_id',
        urlRoot: '/application/like/'
    });

    AppView.collections.LikesCollection = Backbone.Collection.extend({
        model: AppView.models.LikesModel,

        url: function() {
            return 'application/likes/' + this.share_id
        },

        initialize: function (models, options) {
            this.share_id = options.share_id;
            this.url = this.url();
        },

        parse: function(data) {
            return data.likes;
        },

        like: function() {
            //cast id to a string and look inside of a collection for specified model
            var model = this.findWhere({user_id: "" + user.id});
            if (model === undefined) {
                var model = new AppView.models.LikesModel({
                    share_id: this.share_id,
                    user_id: user.id,
                    user_name: user.email
                });
                this.add(model);
                model.save();
            } else {
                this.remove(model);
                model.destroy();
            }
        }
    });

    AppView.views.LikesViewList = Backbone.View.extend({
        collection: AppView.collections.LikesCollection,
        className: 'post-likes-list',
        tagName: 'span',

        events: {
            'mouseenter a.post-like-button': 'fetch',
            'mouseleave a.post-like-button': 'hideLikes',
            'click a.post-like-button': 'like',
            'click a.post-liked': 'dislike'
        },

        initialize: function() {
            //listen to any addition of new like
            this.listenTo(this.collection, 'add', this.add);
            this.listenTo(this.collection, 'reset', this.remove);
            this.listenTo(this.collection, 'remove', this.remove);
            this.listenTo(this.collection, 'destroy', this.remove);
            this.$likesList = this.$el.find('.post-like-likes');
            this.$likesCount = this.$el.find('.post-likes-count');
        },

        add: function(data) {
            var view = new AppView.views.LikeView({model: data});
            view.render();
            this.$likesList.show().append(view.el);
            this.$likesCount.show();
            this.updateCounter();
        },

        remove: function() {
            this.render();
            this.updateCounter();
        },

        render: function() {
            this.$likesList.html('');
            for (var i = 0, len = this.collection.length; i < len; ++i) {
                this.add(this.collection.models[i]);
            }

            return this;
        },

        hideLikes: function()  {
            this.$likesList.hide();
        },

        fetch: function() {
            var self = this;
            this.collection.fetch({
                success: function() {
                    self.$likesCount[!self.collection.length ? 'hide' : 'show']();
                    self.updateCounter();
                    if (self.collection.length)
                        self.$likesList.show();
                }
            });
        },

        like: function() {
            this.collection.like();
        },

        updateCounter: function () {
            this.$likesCount.text('(' + this.collection.length + ')');
        }
    });

    AppView.views.LikeView = Backbone.View.extend({
        model: AppView.models.LikesModel,
        template: _.template(
            '<div><%- user_name %></div>'
        ),

        initialize: function() {
            //if a model is destroyed, then view should be also
            this.listenTo(this.model, 'destroy', this.remove);
        },

        render: function () {
            this.$el.html(this.template(this.model.attributes));
        },

        remove: function() {
            //remove el from DOM
            this.$el.remove();
        }
    });
Let's analyze this code a little.

Each like is an single user info. 
    
    :::javascript
    '<div><%- user_name %></div>'
By adding to model
    
    :::javascript
    AppView.models.LikesModel = Backbone.Model.extend({
        idAttribute: 'like_id',
        urlRoot: '/application/like/'
    });
**urlRoot** backbone allows now to use save function. When this function will be called an **AJAX POST** request will be made to this URL. All data set to the model will also be sent to the server. **Really important** thing is that the data will be sent as a json inside of a **POST body**, not as a parameters! You will have to read body of request and parse it.

To update/delete already existing *Like* Backbone needs to know which *Like* exactly it shoudl update. For that it needs a resource identifier (in our case it will be primary key for id)

Depending if the like model is already inside collection of not we want to like a message or unlike it. Backbone gives possibility to query collection using ```findWhere``` method.

    :::javascript
    like: function() {
        var model = this.findWhere({user_id: "" + user.id});
        if (model === undefined) {
            var model = new AppView.models.LikesModel({
                share_id: this.share_id,
                user_id: user.id,
                user_name: user.email
            });
            this.add(model);
            model.save();
        } else {
            this.remove(model);
            model.destroy();
        }
    }
In ```LikesCollection``` there's a dynamic URL specified based on ```share_id```. There's similar trick done (like previously with attributes) with definition of ```url``` attribute

    :::javascript
    url: function() {
        return 'application/likes/' + this.share_id
    },
Again like last time, if url would be a string, then value would be set before creating an instance (during the parse time). This would lead to have exactly the same ```share_id``` across all instances, which is not what we expect.

To make likes working, you have to also update initialize method of  ```ShareView``` class

    :::javascript
    initialize: function(attr) {
        this.listenTo(this.model, 'change', this.render);
        this.listenTo(this.model, 'destroy', this.remove);

        this.render();
        this.likesList = new AppView.views.LikesViewList({
            el: this.$el.find('.post-like'),
            //initialize collection but with no models for now
            collection: new AppView.collections.LikesCollection([], {
                share_id: this.model.get('share_id')
            })
        });
        this.likesList.render();
    },

### Routing
For now it won't still work, because we didn't put any code on the server side to create/remove a like. Firstly let's configure our module further by adding new routes and invokable. Modify ```modules/Application/config/module.config.php```

    :::php
    <?php
    return array(
        //...
        'routes' => array(
            //...
            'application' => array(
                //...
                'child_routes' => array(
                    likes' => array(
                        'type' => 'Segment',
                        'options' => array(
                            'route' => '/likes/:shareId',
                            'constraints' => array(
                                'shareId' => '[0-9]+',
                            ),
                            'defaults' => array(
                                '__NAMESPACE__' => 'Application\Controller',
                                'controller'    => 'Like',
                                'action'        => 'index',
                            ),
                        ),
                    ),
                    'like' => array(
                        'type' => 'Segment',
                        'options' => array(
                            'route' => '/like/[:likeId]',
                            'constraints' => array(
                                'likeId' => '[0-9]+',
                            ),
                            'defaults' => array(
                                '__NAMESPACE__' => 'Application\Controller',
                                'controller'    => 'Like',
                                'action'        => 'edit',
                            ),
                        ),
                    ),
                    'default' => array(
                        //....
                    ),
                ),
            ),
        ),
        //...
        'controllers' => array(
            'invokables' => array(
                'Application\Controller\Index' => 'Application\Controller\IndexController',
                'Application\Controller\Share' => 'Application\Controller\ShareController',
                'Application\Controller\Like' => 'Application\Controller\LikeController',
            ),
        ),
    );

### Managing Likes
Let's create ```LikeController```, so now storing new likes should be possible

    :::php
    <?php

    namespace Application\Controller;

    use Zend\Mvc\Controller\AbstractActionController;
    use Zend\View\Model\ViewModel;
    use Zend\Http\Request;
    use Zend\Json\Json;

    use Application\Form;
    use Application\Model;

    class LikeController extends AbstractActionController
    {
        protected $likesTable;

        protected $shareTable;

        protected $acceptCriteria = array(
            'Zend\View\Model\JsonModel' => array(
                'application/json',
            ),
        );

        protected function getLikesTable()
        {
            if ($this->likesTable !== null) {
                return $this->likesTable;
            }

            $sm = $this->getServiceLocator();
            return $this->likesTable = $sm->get('Application\Model\LikesTable');
        }

        public function getShareTable()
        {
            if (!$this->shareTable) {
                $sm = $this->getServiceLocator();
                $this->shareTable = $sm->get('Application\Model\ShareTable');
            }

            return $this->shareTable;
        }

        public function indexAction()
        {
            $viewModel = $this->acceptableViewModelSelector($this->acceptCriteria);
            $likesTable = $this->getLikesTable();

            $viewModel->setVariables(array(
                'likes' => $likesTable->getByShareId(
                        $this->params()->fromRoute('shareId')
                    ),
                )
            );
            return $viewModel;
        }

        public function editAction()
        {
            $viewModel = $this->acceptableViewModelSelector($this->acceptCriteria);
            $likesTable = $this->getLikesTable();
            $shareTable = $this->getShareTable();
            $like = current($likesTable->findByLikeId($this->params()->fromRoute('likeId'))->toArray());
            if ($this->getRequest()->getMethod() == Request::METHOD_DELETE) {

                $likesTable->deleteByLikeUserId(
                    $this->params()->fromRoute('likeId'),
                    $this->zfcUserAuthentication()->getIdentity()->getId(),
                    $like['share_id'],
                    $shareTable
                );
            } elseif ($this->getRequest()->getMethod() === Request::METHOD_POST) {
                $content = $this->getRequest()->getContent();
                $likesTable->add(Json::decode($content, Json::TYPE_ARRAY), $shareTable);
            }
            return $viewModel;
        }
    }
As you can see there's only one action for adding and removing a Like. This is because backbone sends AJAX request to the same URL defined in ```urlRoot``` attribute inside a model.

Now you should be able to see likes list while hovering above them with cursor and add or remove them by clicking on like button.

## TL;DR (or summarization)
**Backbone.js** is a nice framework to organize your JavaScript code. It also nicely connects with **Zend Framework 2** thought it requires some work. But remember **Backbone.js** shouldn't be used in every project! You should only use it in case when you have a huge JS codebase. In other case it is a waste of transfer to add **Backbone.js and Underscore**.

As a big minus of this Framework should be noted that there's a lot of boilerplate code necessary to write an application. It's also not sure where to put some pieces of code. Should the share handle likes, or should like be put inside of separate view? This can still lead to spaghetti code if you're not cautious.

Then why did I use **Backbone**? True, that there are other frameworks for JavaScript like previously noted **Angular** or **Ember**, still I wanted to be able to generate some of the content strictly server side and send it to a browser. This is where **Backbone** is a win above Angular or Ember. If this fact is not important for you, then stick to **Angular**, where you will write less code and do more!

