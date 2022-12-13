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

  - title: API for the “Do-it-yourself" system
    image_path: building_block.png  
    excerpt: >-
      Bring your own Protocol or Controller – Use the same APIs that the IOS-XR protocol stacks use internally, but over GRPC.

  - title: Offload Low-level tasks to IOS-XR
    image_path: offload.png
    excerpt: >-
      Users can focus on higher layer protocols and Controller logic while the IOS-XR infrastructure layer handles conflict resolution, transactional notifications, scalability and data plane abstraction.

published: true
---
{% include base_path %} 

{% include feature_row id="feature_row_benefits" %}

<div class="feature__wrapper">
    <div class="feature__item--right">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="display: block; margin-left: auto; margin-right: auto;">
            <a href="https://xrdocs.io/images/openr-integration-xr.png"><img src="https://xrdocs.io/images/openr-integration-xr.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="https://xrdocs.github.io/cisco-service-layer/blogs/2018-02-16-xr-s-journey-to-the-we-b-st-open-r-integration-with-ios-xr/">Open/R integration with IOS XR</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Explore how IOS-XR’s service layer APIs and application hosting capabilities can be
                                leveraged to host and integrate Open/R as an IGP on IOS-XR.</p>
            </div>
          <p><a href="https://xrdocs.github.io/cisco-service-layer/blogs/2018-02-16-xr-s-journey-to-the-we-b-st-open-r-integration-with-ios-xr/" 
          class="btn ">Learn more</a></p>        </div>
      </div>
    </div>
</div>

<div class="feature__wrapper">    
    <div class="feature__item--left">
      <div class="archive__item" style="margin-left: 2em;">
          <div class="archive__item-teaser center" style="display: block; margin-left: auto; margin-right: auto;">
              <iframe width="560" height="315" src="https://www.youtube.com/embed/yiN8fAxlfwg" frameborder="0" allowfullscreen></iframe>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="https://www.youtube.com/watch?v=yiN8fAxlfwg"></a>Techfield Day 15 @Cisco Systems</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
            <p>Akshat Sharma gives a lowdown on the latest advancements in Cisco IOS-XR programmability as Cisco enables end-users to take advantange of a high performance channel to directly program the IOS-XR infrastructure layer, called the Service Layer, over gRPC. The talk focuses on the architectural tenets of Service Layer APIs and ends with a demo of an OpenBMP controller implemented using the Service Layer APIs.</p>
            </div>
          <p><a href="https://www.youtube.com/watch?v=yiN8fAxlfwg" class="btn btn--large">Watch on Youtube</a></p>
        </div>
      </div>
    </div>
</div>

<div class="feature__wrapper">
    <div class="feature__item--right">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 300px; max-width: 300px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/apidocs"><img src="{{ base_path }}/images/apidocs.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="{{ base_path }}/apidocs">Documentation auto-generated from Code</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Doxygen Based API Documentation auto-generated from the protobuf IDLs. Documentation always remains up to date with the API!</p>
            </div>
          <p><a href="{{ base_path }}/apidocs" class="btn ">Check out the APIdocs</a></p>
        </div>
      </div>
    </div>
</div>

<div class="feature__wrapper">
    <div class="feature__item--right">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/tutorials/2017-09-26-use-case-openbmp-controller-using-service-layer-apis/"><img src="{{ base_path  }}/images/openbmp-controller.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title"><a href="{{ base_path }}/tutorials/2017-09-26-use-case-openbmp-controller-using-service-layer-apis/">OpenBMP Controller using Service Layer APIs</a></h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>Selective Route download for BGP is a use case that most CDN network operators have taken a stab at. Being able to manipulate the RIB directly with custom route policies to optimize TCAM usage at a CDN PoP router is made much simpler through a high performance Model-Driven API directly into the XR RIB as part of the Cisco Service Layer. <b>Akshat Sharma</b> explains.</p>
            </div>
          <p><a href="{{ base_path }}/tutorials/2017-09-26-use-case-openbmp-controller-using-service-layer-apis/" class="btn ">Take a Look</a></p>
        </div>
      </div>
    </div>
</div>
