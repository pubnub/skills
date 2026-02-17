# Serverless Product Catalog API

## Problem/Feature Description

Our e-commerce team needs a lightweight serverless API to manage a small product catalog. The API will be consumed by a single-page web application running in the browser, so cross-origin requests need to be handled properly. The API should support creating, reading, updating, and deleting products using standard HTTP methods. Products should be stored persistently with a reasonable expiration time. Each product has an `id`, `name`, `price`, and `category`.

The API should be built as a PubNub Function that handles incoming HTTP requests and performs CRUD operations against persistent storage. The web frontend will call this endpoint directly from the browser.

## Output Specification

Produce the following files:

1. **`product-api.js`** -- The complete PubNub Function source code for the HTTP endpoint.
2. **`deployment.md`** -- Instructions covering how to deploy the function, including what type of function trigger to select, and any necessary setup steps.
