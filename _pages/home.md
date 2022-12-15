---
layout: homepage
permalink: /
sitemap: true
date: null
excerpt: High performance access to the IOS-XR infrastructure Layer including RIB, Label Switch Database and more. Bring your own protocol or controller and operate your network your way!

feature_row_benefits:
  - title: Containers 
    image_path: container_networking.png
    excerpt: >-
      XRd is a family of container-based IOSXR-LNT platforms. It is a lightweight solution that can be used as a vRR (virtual-route-reflector), provider edge, vCSR (virtual Cell-Site Router), and cloud router (gateway for the cloud).

  - title: Cloud Deployment
    image_path: container_cloud.png  
    excerpt: >-
      Both XRd (container) and XRv9k (VM) function in cloud environments; with current support for AWS.
  
  - title: Kubernetes Orchestration
    image_path: Kubernetes_logo.png  
    excerpt: >-
      Automate the orchestration and deployment of XRd containers with Kubernetes.
published: true
---
{% include base_path %} 

{% include feature_row id="feature_row_benefits" %}

<div class="feature__wrapper">
    <div class="feature__item--right">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 300px; max-width: 300px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/tutorials/2022-08-22-xrd-images-where-can-one-get-them/><img src="{{ base_path }}/images/docker-iosxr.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="{{ base_path }}/tutorials/2022-08-22-xrd-images-where-can-one-get-them/">Get Started with XRd</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Download the latest XRd image and learn how to set up your environment to deploy it locally</p>
            </div>
          <p><a href="{{ base_path }}//tutorials/2022-08-22-xrd-images-where-can-one-get-them/" class="btn ">Check out XRd tutorials</a></p>
        </div>
      </div>
    </div>
</div>

<div class="feature__wrapper">
    <div class="feature__item--left">
      <div class="archive__item" style="margin-left: 2em;">
          <div class="archive__item-teaser center" style="max-height: 200px; max-width: 200px; display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/tutorials/2022-12-08-deploy-xrd-on-aws/"><img src="{{ base_path  }}/images/aws-eks-logo.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="{{ base_path }}/tutorials/2022-12-08-deploy-xrd-on-aws/">Deploy XRd on AWS</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Launch XRd on AWS, by deploying XRd on an EKS cluster</p>
            </div>
          <p><a href="{{ base_path }}/tutorials/2022-12-08-deploy-xrd-on-aws/" class="btn ">Take a Look</a></p>
        </div>
      </div>
    </div>
</div>
<div class="feature__wrapper">
    <div class="feature__item--right">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 300px; max-width: 300px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/blogs/2022-08-23-high-availability-with-xrv9000-on-aws-using-apphosting-sl-api/"><img src="{{ base_path }}/images/HA-xrv9k.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="{{ base_path }}/blogs/2022-08-23-high-availability-with-xrv9000-on-aws-using-apphosting-sl-api/">High Availability with XRv9000 on AWS: Using AppHosting + SL-API</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Learn how to enable 1:1 high-availability with XRv9000 when running on AWS. In this blog we discuss a solution to achieve HA by building our own third-party container application that hooks into IOS-XR Service-layer APIs and can be deployed using XR AppMgr</p>
            </div>
          <p><a href="{{ base_path }}/blogs/2022-08-23-high-availability-with-xrv9000-on-aws-using-apphosting-sl-api/" class="btn ">Learn More</a></p>
        </div>
      </div>
    </div>
