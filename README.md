# Authentication, Identity, and RBAC for Zend Framework 3
[![Build Status](https://travis-ci.org/Saeven/zf3-circlical-user.svg?branch=master)](https://travis-ci.org/Saeven/zf3-circlical-user)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/fe24b2bf7ab74919844fdb49adbf99fe)](https://www.codacy.com/app/alemaire/zf3-circlical-user?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Saeven/zf3-circlical-user&amp;utm_campaign=Badge_Grade)
[![Codacy Badge](https://api.codacy.com/project/badge/Coverage/fe24b2bf7ab74919844fdb49adbf99fe)](https://www.codacy.com/app/alemaire/zf3-circlical-user?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Saeven/zf3-circlical-user&amp;utm_campaign=Badge_Coverage)
[![Latest Stable Version](https://poser.pugx.org/saeven/zf3-circlical-user/v/stable)](https://packagist.org/packages/saeven/zf3-circlical-user)
[![Total Downloads](https://poser.pugx.org/saeven/zf3-circlical-user/downloads)](https://packagist.org/packages/saeven/zf3-circlical-user)

Plug and play authentication, roles, resource, and action control for Zend Framework 3.

Quickly Installs:

- cookie based authentication (using halite and its authenticated encryption)
- role-based access control (RBAC) with guards at the controller and action level
- user-based access control to complement RBAC 
- resource-based permissions, giving you 'resource' and 'verb' control at the role and user level, e.g. (all administrators can 'add' a server, only Pete can 'delete')

### Missive

Sure - there are other Authentication, ACL, and User modules out there.  This one comes with out-of-the-box support for **Doctrine** - just plug in your user entity and go.

Authentication is persisted using cookies, meaning no session usage at all.  This was done because I develop for 
circumstances where this is preferable, removing any need for complex or error-prone solutions for session management on 
an EC2 auto-scale group for example.

Lastly, authenticated encryption is handled using the well-trusted [Halite](https://github.com/paragonie/halite), and password hashing is 
properly done with PHP's new password functions. 
[Feedback always solicited on r/php.](https://www.reddit.com/r/PHP/comments/4r84jn/need_reviews_of_cookiebased_authentication_service/).
If you are a paranoid fellow like me, this library should serve well!

This library works on a deny-first basis.  Everything defined by its parts below, are 'allow' grants.

## User Authentication

The module provides full identity/auth management, starting at the user-level.  A design goal was to connect this to 
registration or login processes with little more than one-liners.

#### Login

Validate your submitted Login form, and then execute this to get your user through the door:
       
    $user = $this->auth()->authenticate( $emailOrUsername, $password );
    
Successful authentication, will drop cookies that satisfy subsequent identity retrieval.
     
     
#### Logout
    
Trash cookies and regenerate the session key for that user, using this command:
    
     $this->auth()->clearIdentity();


## Pluggable Deny Strategy

Someone trying to do something they shouldn't? It's easy to control what happens with a pluggable DenyStrategy.  Create a class
that implements DenyStrategyInterface and plug it into your config.  This module comes with a default **RedirectStrategy** that will
send users to a login page, if the problem was that there was no auth, and it wasn't an XHTTP request.  Easy to use, you'd
configure it like so:

    'deny_strategy' => [

        'class' => \CirclicalUser\Strategy\RedirectStrategy::class,

        'options' => [
            'controller' => \Application\Controller\LoginController::class,
            'action' => 'index',
        ],
    ],

Writing your own should be very simple, see provided tests.
     
     
## Creating Access For Your Users
     
Your app needs to be modified to create a distinct auth record for each user.  It's very simple.


#### create & authenticate
During user registration routines, you probably want to create the records and also log them in.  To accomplish this, you can use the helper or the 'create' method
on AccessService.

From a Controller, you can use the auth plugin:
     
     $this->auth()->create(User $user, string $usernameOrEmail, string $password); // controller helper

or, the AuthenticationService:

    $container->get(AuthenticationService::class)->create($user, $usernameOrEmail, $password);
    
#### create only    
Otherwise, if you simply want to create a user auth record but not log them in, use:

    $container->get(AuthenticationService::class)->registerAuthenticationRecord(User $user, string $username, string $password)


## Roles

Your users belong to hierarchical roles that are configured in the database.  *The default guest user, is group-less.*  
Roles are used to restrict access to **controllers**, **actions**, or **resources**.


## Guards

Guards are conditions on **controllers** or **actions** that examine **group** or **user** privileges to permit/decline 
attempted access.  It works very similarly to [BjyAuthorize](https://github.com/bjyoungblood/BjyAuthorize) 
(a great module I used for years).

Configuring guards is very simple.  Your module's config would look like so:

     return [
        'circlical' => [
            'user' => [
                'guards' => [
                    'ModuleName' => [
                        "controllers" => [
                            \Application\Controller\IndexController::class => [
                                'default' => [], // anyone can access
                            ],
                            \Application\Controller\MemberController::class => [
                                'default' => ['user'], // specific role access
                            ],
                            \Application\Controller\AdminController::class => [
                                'default' => ['admin'],
                                'actions' => [  // action-level guards
                                    'list' => [ 'user' ], // role 'user' can access 'listAction' on AdminController
                                ],
                            ],
                        ],
                    ],
                ],
            ],
        ],
     ];   


## Resources & Permissions

Resources can be:

* simple strings, or
* anything you have that implements [ResourceInterface](src/CirclicalUser/Provider/ResourceInterface.php)
  
Both these usages are valid from a controller:

    $this->auth()->isAllowed('door','open');
    
or if an object:
    
    // server implements ResourceInterface
    $server = $serverMapper->get(142);
    $this->auth()->isAllowed($server,'shutdown');

The AccessService is also similarly usable.  See [AccessService tests](bundle/Spec/Service/AccessServiceSpec.php) for more usage examples.
    
Granting a **role** a **permission** is done through the AccessService 

### User Permissions

You can also give individual users, access to specific actions on resources as well.  This library provides 
 **Doctrine** entities and a mapper to make this happen -- but you could wire your own [UserPermissionProviderInterface](src/CirclicalUser/Provider/UserPermissionProviderInterface.php)
 very easily.  In short, this lets the AccessService use the authenticated user to determine whether or
 not the logged-in individual can perform an action that supersedes what his role permissions otherwise
 grant.  User Permissions are meant to be more permissive, not restrictive.  


# Installation

* Working with Doctrine? [Click Here](INSTALL_DOCTRINE.md) 
* Want to roll your own Providers? [Click Here](INSTALL_CUSTOM.md)

