---
layout: post
title: "Decorate AngularJS $http service to convert PUT, DELETE requests to POST"
date: 2013-07-25 10:46
comments: true
categories: [code, javascript, angularjs, jersey]
description: "Decorate AngularJS $http service to convert PUT, DELETE requests to POST" 
keywords: "code, javascript, angularjs, jersey"
---
Below is the code for AngularJS 1.0.x to convert `PUT` and `DELETE` requests to 'POST' over the network, but use them as they were. AngularJS 1.0.x has Responses Interceptors but no Request Interceptors (available from 1.1.x). So, I came with this code and of course it's not pretty :)

``` javascript 
/** AngularJS $http service decorator to convert PUT, DELETE as POST */
var App = angular.module('App');
App.config(['$provide', function($provide) {
	
	// configure http provider to convert 'PUT', 'DELETE' methods to 'POST' requests
	$provide.decorator('$http', ['$delegate', function($http) {
		// create function which overrides $http function
		var httpStub = function (method) {
			return function(url, data, config) {
				// AngularJS $http.delete takes 2nd argument as 'config' object
				// 'data' will come as 'config.data'
				if(method === 'delete') {
					config = data;
					config && (data = config.data);
				}
				
				config || (config = {});
				config.headers || (config.headers = {});
				
				// override actual request method with 'POST' request
				config.method = 'POST';
				
				// set the actual method in the header
				config.headers['X-HTTP-Method-Override'] = method;
				
				return $http(angular.extend(config, {
					url: url,
					data: data
				}));
			};
		};
		
		// backup of original methods
		$http._put = $http.put;
		$http._delete = $http['delete'];
		
		// override the 
		$http.put = httpStub('put');
		$http['delete'] = httpStub('delete');
		
		return $http;
	}]);
	
}]);
```

### Why is it required? ###

I use REST (JAX-RS) approach for developing & exposing server side services are resources in almost all projects and I totally recommend it. 

<!-- more -->

And, REST recommends to use appropriate HTTP method designators (*ex*: `GET` to get all user names, `POST` to save a user details). But sometimes for some projects, *especially,* in production environments this recommendation is undesirable as few things allowing HTTP methods other than `GET` and `POST` (`PUT`, `DELETE` and other methods) will degrade the server security and I m not sure this should be a concern now a days if we correctly configure the front-end server, *like Apache*, or any solid reason?

We faced the same scenario in current project in which we use the excellent [AngularJS](http://angularjs.org) framework for building client side UI, and [Jersey](http://jersey.java.net/) as JAX-RS implementation.

But, I like the idea of using the appropriate HTTP methods instead of using `POST` for all requests but with different URI, *like* `/user/add` for saving user details, and `user/delete` for deleting a user, which is totally out of sync with REST idea. My idea is to use the semantically correct methods in the code, and to have a client & server side converters to the necessary conversions from `PUT` and `DELETE` to `POST` method.   

Jersey out of box provides `PostReplaceFilter` which treats `POST` request with a special header or query param as `PUT` or `DELETE` bases on the header/param value.

``` xml web.xml (Jersey POST replacement container request filter)
		<!-- support HTTP method replacing of a POST request to be firewall friendly; actual request method 
			will be set either in 'X-HTTP-Method-Override', or a query parameter '_method' -->
		<init-param>
			<param-name>com.sun.jersey.spi.container.ContainerRequestFilters</param-name>
			<param-value>com.sun.jersey.api.container.filter.PostReplaceFilter</param-value>
		</init-param>
```

But to a achieve required functionality in AngularJS, *especially in v1.0.x*, there is no Request Interceptor concept :( So I had to come up with the above $http service decorator, which is not that difficult to write.
