<p align="center">
  <img src="https://res.cloudinary.com/dqbgnn5hf/image/upload/c_scale,w_200/v1610530093/padlock.svg" width="200" height="200">
</p>

# Secure your backend with aws

While i was working on a react project recently and had to use an api to fetch data for the application i faced an issue while i was trying to deploy the app on Netlify for testing and feedback, the api endpoint was not secured (https) which caused security issues. I wanted an easy solution to be able to easily secure any endpoint so i created a small cli tool to solve this.

In this blog post i will explain how i wrote the cli tool and show you how to simply use aws’s gateway to secure your endpoints.

## Usage

You need to create an aws account (if you don’t have one already)

    npm install --save @walidelnozahy/secure

then simply just call `secure http://example.com` in your terminal

    secure http://example.com

    // https://id.execute-api.us-west-1.amazonaws.com/dev

## How it works

Steps needed to secure the endpoint with api gateway are:

    1. Create new rest api.
    2. Get resources for the created rest api (to get parent id).
    3. Create resource (with parent id).
    4. Put method with created resource id and request params.
    5. Put method with parent id.
    6. Put integration for the resource id.
    7. Put integration for the parent id.
    8. Create deployment.
    9. Create stage.
