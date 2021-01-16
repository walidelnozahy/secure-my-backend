Secure your backend with aws’s api gateway
While i was working on a react project recently and had to use an api to fetch data for the application i faced an issue while i was trying to deploy the app on Netlify for testing and feedback, the api endpoint was not secured (https) which lead to some CORS problems. I wanted an easy solution to be able to easily secure any endpoint so i created a small cli tool to solve this.

In this blog post i will explain how i wrote the cli tool and show you how to simply use aws’s gateway to secure your endpoints.

Usage

You need to create an aws account (if you don’t have one already)

    npm install --save @walidelnozahy/secure

then simply just call `secure http://example.com` in your terminal

    secure http://example.com

    // https://id.execute-api.us-west-1.amazonaws.com/dev

How it works

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

so let’s get started. first we will need aws’s sdk
Install aws-sdk by :

    npm install --save aws-sdk

then we’ll create a file called `secure.js`.

we will require `aws` and update the region in the config for the desired region in this case we will use `us-west-1` , then create a function called `secureMyBackend` that will initiate everything

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const secureMyBackend = async () => {}

Creating the Rest API

first we will need to create a rest api and give it a `name` and `endpointConfiguration` with `types` of ` [``'``REGIONAL``'``] ` and then get it’s id because we will need it later, and then we will get resources for rest api that we just created in order to get the `parentId`.

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const createRestApi = async (name) => {
      const res = await apigateway
        .createRestApi({
          name,
          endpointConfiguration: {
            types: ['REGIONAL']
          }
        })
        .promise()
      return res.id
    }

    const secureMyBackend = async () => {
      const restApiId = await createRestApi(`new api`)
     const resources = await apigateway
        .getResources({
          restApiId
        })
        .promise()
    const parentId = resources.items[0].id
    }

Creating Resources & Methods

then we need to create resources and methods.

We will create the resources using `createResource` and pass `parentId` , `restApId` and the `pathPart` which will take a value of `{proxy+}`

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const createRestApi = async (name) => {
      const res = await apigateway
        .createRestApi({
          name,
          endpointConfiguration: {
            types: ['REGIONAL']
          }
        })
        .promise()
      return res.id
    }

    const createResource = async ({ restApiId, parentId }) => {
      const res = await apigateway
        .createResource({
          parentId,
          pathPart: '{proxy+}',
          restApiId
        })
        .promise()
      return res.id
    }


    const secureMyBackend = async () => {
      const restApiId = await createRestApi(`new api`)
     const resources = await apigateway
        .getResources({
          restApiId
        })
        .promise()
    const parentId = resources.items[0].id
    const resourceId = await createResource({ restApiId, parentId })
    }

then we need to put two methods, one for the resource we just created and pass `requestParameters` with value `{` ` '``integration.request.path.proxy``'``: ` ` '``method.request.path.proxy``'``} ` and another put method for the resource created with `restApiId` and pass `parentId`

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const createRestApi = async (name) => {
      const res = await apigateway
        .createRestApi({
          name,
          endpointConfiguration: {
            types: ['REGIONAL']
          }
        })
        .promise()
      return res.id
    }

    const createResource = async ({ restApiId, parentId }) => {
      const res = await apigateway
        .createResource({
          parentId,
          pathPart: '{proxy+}',
          restApiId
        })
        .promise()
      return res.id
    }

    const putMethod = async ({ resourceId, restApiId, requestParameters }) => {
      const res = await apigateway
        .putMethod({
          ...({ requestParameters } || {}),
          authorizationType: 'NONE',
          httpMethod: 'ANY',
          resourceId,
          restApiId
        })
        .promise()
      return res
    }

    const secureMyBackend = async () => {
      const restApiId = await createRestApi(`new api`)
     const resources = await apigateway
        .getResources({
          restApiId
        })
        .promise()
    const parentId = resources.items[0].id
    const resourceId = await createResource({ restApiId, parentId })
    await putMethod({
        restApiId,
        resourceId,
        requestParameters: { 'method.request.path.proxy': true }
      })
      await putMethod({ restApiId, resourceId: parentId })
    }

Creating the Proxy Integration

we need to put 2 integrations one for `resourceId` and pass `requestParameters` as `'integration.request.path.proxy': 'method.request.path.proxy'` and pass ` uri: 'url/{proxy}``' ` and one for `parentId` and pass the uri

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const createRestApi = async (name) => {
      const res = await apigateway
        .createRestApi({
          name,
          endpointConfiguration: {
            types: ['REGIONAL']
          }
        })
        .promise()
      return res.id
    }

    const createResource = async ({ restApiId, parentId }) => {
      const res = await apigateway
        .createResource({
          parentId,
          pathPart: '{proxy+}',
          restApiId
        })
        .promise()
      return res.id
    }

    const putMethod = async ({ resourceId, restApiId, requestParameters }) => {
      const res = await apigateway
        .putMethod({
          ...({ requestParameters } || {}),
          authorizationType: 'NONE',
          httpMethod: 'ANY',
          resourceId,
          restApiId
        })
        .promise()
      return res
    }

    const putIntegration = async (props) => {
      const { resourceId, restApiId, uri, requestParameters } = props
      const params = {
        httpMethod: 'ANY',
        integrationHttpMethod: 'ANY',
        type: 'HTTP_PROXY',
        resourceId,
        restApiId,
        uri,
        ...({ requestParameters } || {}),
        passthroughBehavior: 'WHEN_NO_MATCH'
      }
      const res = await apigateway.putIntegration(params).promise()
      return res
    }

    const secureMyBackend = async () => {
      const restApiId = await createRestApi(`new api`)
     const resources = await apigateway
        .getResources({
          restApiId
        })
        .promise()
    const parentId = resources.items[0].id
    const resourceId = await createResource({ restApiId, parentId })
    await putMethod({
        restApiId,
        resourceId,
        requestParameters: { 'method.request.path.proxy': true }
      })
      await putMethod({ restApiId, resourceId: parentId })
    await putIntegration({
        restApiId,
        resourceId,
        uri: `${url}/{proxy}`,
        requestParameters: {
          'integration.request.path.proxy': 'method.request.path.proxy'
        }
      })
      await putIntegration({
        restApiId,
        resourceId: parentId,
        uri: url
      })
    }

Deploying the API

and finally we need to create deployment and create stage with `deploymentId` and then return the new secured endpoint

    // secure.js
    const aws = require('aws-sdk')
    aws.config.update({ region: 'us-west-1' })

    const createRestApi = async (name) => {
      const res = await apigateway
        .createRestApi({
          name,
          endpointConfiguration: {
            types: ['REGIONAL']
          }
        })
        .promise()
      return res.id
    }

    const createResource = async ({ restApiId, parentId }) => {
      const res = await apigateway
        .createResource({
          parentId,
          pathPart: '{proxy+}',
          restApiId
        })
        .promise()
      return res.id
    }

    const putMethod = async ({ resourceId, restApiId, requestParameters }) => {
      const res = await apigateway
        .putMethod({
          ...({ requestParameters } || {}),
          authorizationType: 'NONE',
          httpMethod: 'ANY',
          resourceId,
          restApiId
        })
        .promise()
      return res
    }

    const putIntegration = async (props) => {
      const { resourceId, restApiId, uri, requestParameters } = props
      const params = {
        httpMethod: 'ANY',
        integrationHttpMethod: 'ANY',
        type: 'HTTP_PROXY',
        resourceId,
        restApiId,
        uri,
        ...({ requestParameters } || {}),
        passthroughBehavior: 'WHEN_NO_MATCH'
      }
      const res = await apigateway.putIntegration(params).promise()
      return res
    }

    const secureMyBackend = async () => {
      const restApiId = await createRestApi(`new api`)
     const resources = await apigateway
        .getResources({
          restApiId
        })
        .promise()
    const parentId = resources.items[0].id
    const resourceId = await createResource({ restApiId, parentId })
    await putMethod({
        restApiId,
        resourceId,
        requestParameters: { 'method.request.path.proxy': true }
      })
      await putMethod({ restApiId, resourceId: parentId })
    await putIntegration({
        restApiId,
        resourceId,
        uri: `${url}/{proxy}`,
        requestParameters: {
          'integration.request.path.proxy': 'method.request.path.proxy'
        }
      })
      await putIntegration({
        restApiId,
        resourceId: parentId,
        uri: url
      })
      const deploymentId = await createDeployment({ restApiId })
      await createStage({ restApiId, deploymentId })
      return `secured url: https://${restApiId}.execute-api.us-west-1.amazonaws.com/dev`
    }

Conclusion

I hope this short tutorial was helpful. you can find repo of the above example on Github at https://github.com/wellyelnozahy/secure-my-backend

you can find the cli tool here https://www.npmjs.com/package/@walidelnozahy/secure
