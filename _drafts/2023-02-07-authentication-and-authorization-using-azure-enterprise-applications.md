---
title: Authentication and Authorization using Azure Enterprise Applications
categories: [azure]
tags: [azure, authentication, authorization]
---

## Azure SSO is cool

Have you ever seen one of these login forms?

![Azure Oauth Sign-in](/assets/img/azure-sso-signin.png)
_Microsoft Azure Single Sign-on_

Single sign-on (SSO) is a pretty slick feature you can leverage if you make use of Azure Active Directoy. While I might write an article later about how to set this up, I want to explain something significant about Azure's SSO. 

Let's talk about the difference between *authentication* and *authorization*.

## Authentication vs. Authorization

When you sign into one of these SSO forms, quite a few things happen.

## Authentication

This step simply confirms that you are who you say you are. If I supply the correct username and password for my account, the authentication step should just say, "yep, let him in".

## Authorization

After I'm confirmed to be me, the next step is checking if I have permission to access the underlying resource. By default in Azure SSO, any user that can authenticate can access the resource behind it. This means that you should handle permissions and role-based access control in your application, which is typically the correct thing to do.

### Jupyterhub

Jupyterhub is a platform where authorized users are presented with containerized notebooks. You can configure various types of *authentication*, and we've opted to use Azure SSO. The user navigates to the Jupyterhub cluster, is presented with an Azure SSO prompt, logs in, and their username attribute is passed back to the cluster. Jupyterhub now compares that username to its list of *authorized* users, and allows or denies access to the cluster.

The issue with managing users in Jupyterhub is that it requires a modification of the application settings and an application restart. In our case, since we're running this in Kubernetes, it requires managing a list of users in a YAML manifest, then deploying those changes using Helm. 

Since we're not handling any role-based controls, we're simply granting access or not, wouldn't it be nice if we could leverage Azure SSO to handle the *authorization* stage as well?

### Azure Enterprise Applications

The enterprise application is the component in Azure that handles *authentication*, as well as what account attributes are released to the underlying application. Each enterprise application also creates an associated App Registration, which you can think of as a consumer of the enterprise application. The app registration holds configuration like callback settings for when your user authenticates successfully, and token information that you configure your application with, to be able to talk to the enterprise application.

Enterprise applications also have the ability to manage _who has access to the enterprise application_, by modifying a user list. There's even an API to add users to the enterprise application.

So what if we used the enterprise application user list as an authorization source instead of Jupyterhub's user list? We could configure Jupyterhub to simply allow any user that successfully authenticates. Let's do it!

_to be continued_

![Enterprise App](/assets/img/azure-enterprise-app-cloud-application-administrator.png){: w="700" h="400" }
_Image caption_
