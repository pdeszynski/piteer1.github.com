Title: Zend Framework 2 and Doctrine 2 Integration
Category: Misc
Date: 2012-11-18 10:00

Lately I was playing with ZF2 and Doctrine 2, trying to integrate them into one app. There's a nice [**tutorial**](http://www.jasongrimes.org/2012/01/using-doctrine-2-in-zend-framework-2/ "Using Doctrine 2 in Zend Framework 2"). Here I will try to solve the most common problems which are happening during this integration.

##Helper "em"

First of the problems that can occur is the following one:

    [InvalidArgumentException]       
    The helper "em" is not defined.

If you'll have the same error it's most probably caused by using wrong command for ORM generation. By default in Doctrine you should use:
    
    doctrine orm:schema-tool:create

But in case of **ZF2** this command won't work. You should call instead of it: 

    ./vendor/doctrine/doctrine-module/bin/doctrine-module orm:schema-tool:create

##No Metadata Classes to process.

Check if for sure you have correct path in your **module.config.php** setup for **Annotation Driver**, it should look like:

    :::php
    <?php
    namespace MyNamespace;
    //...
    return array(
        //...
        // Doctrine config
        'doctrine' => array(
            'driver' => array(
                __NAMESPACE__ . '_driver' => array(
                    'class' => 'Doctrine\ORM\Mapping\Driver\AnnotationDriver',
                    'cache' => 'array',
                    'paths' => array(__DIR__ . '/../src/' . __NAMESPACE__ . '/Entity')
                ),
                'orm_default' => array(
                    'drivers' => array(
                        __NAMESPACE__ . '\Entity' => __NAMESPACE__ . '_driver'
                    )
                )
            )
        ),
    );

Be sure not to forget the part with namespace!

    :::php
    namespace MyNamespace;

After that you should be able to work with Doctrine!