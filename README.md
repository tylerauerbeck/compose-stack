# Infratographer in docker compose

This repo provides a sample docker compose that stands up the infratographer stack. Currently it includes the following services:

- [Apollo Router]() serving a generated supergraph
- [Node Resolver](https://github.com/infratographer/node-resolver)
- [Tenant API](https://github.com/infratographer/tenant-api)
- [Location API](https://github.com/infratographer/location-api)
- [Metadata API](https://github.com/infratographer/metadata-api)


To start the services you run `docker compose up` and then you can access the apollo router on port 4000. Loading this
in your browser will show you the Apollo sandbox which allows you to run queries and see the GraphQL schema that is
supported.

## Known issues

- Metadata API doesn't disable authn when in dev mode, so requests to metadata will require a valid JWT


## Examples to get started

### Create a root tenant

Most objects within Infratographer are owners by a `ResourceOwner`. Out of the box we provide Tenants as an implementation
of `ResourceOwner`. To create a tenant run the following operation:

```
mutation {
  tenantCreate(input: {
    name: "My Test Tenant"
    description: "compose-stack walk through root tenant"
  }) {
    tenant {
      id
      name
    }
  }
}
```

You should receive a response similar to:

```
{
  "data": {
    "tenantCreate": {
      "tenant": {
        "id": "tnntten-DPXRgDGEaYBIffcfjp-Po",
        "name": "My Test Tenant"
      }
    }
  }
}
```

With a different ID. Make a note of that ID as we will be using it in the next command.

### Create a child tenant

Tenants are able to have children tenants, which provides a way to organize the resources you deploy in your account.
Let's go ahead and create a child tenant now.

```
mutation {
  tenantCreate(input: {
    name: "Development"
    description: "Tenant for dev resources",
    parentID: "tnntten-DPXRgDGEaYBIffcfjp-Po"
  }) {
    tenant {
      id
      name
    }
  }
}
```

You should receive a response similar to:

```
{
  "data": {
    "tenantCreate": {
      "tenant": {
        "id": "tnntten-rXirlFQULBHDw9urtOjya",
        "name": "Development"
      }
    }
  }
}
```

### Create a location

Locations have to belong to a resource owner, so using a tenant ID from above you can create a location.

```
mutation {
  locationCreate(input: {
    name: "Just a test location"
    description: "location to show creating locations"
    ownerID: "tnntten-rXirlFQULBHDw9urtOjya"
  }) {
    location {
      id
      name
      owner {
        id
      }
    }
  }
}
```

You shoudl receive a response similiar to:

```
{
  "data": {
    "locationCreate": {
      "location": {
        "id": "lctnloc-sEdikPSFsjYJjJxssdTkY",
        "name": "Just a test location",
        "owner": {
          "id": "tnntten-rXirlFQULBHDw9urtOjya"
        }
      }
    }
  }
}
```


### See it all together

Now lets say we want to seeeverything we just created. If we query for our original tenant that we created and ask for the information about that tenant, including it's locations, as well as all of our children tenants and their locations, we can do that with this query:

```
query {
  tenant(id: "tnntten-DPXRgDGEaYBIffcfjp-Po") {
    id
    name
    description
    children {
      edges {
        node {
          id
          name
          description
          locations {
            edges {
              node {
                id
                name
                description
              }
            }
          }
        }
      }
    }
    locations {
      edges {
        node {
          id
          name
          description
        }
      }
    }
  }
}
```

The response should look something like:

```
{
  "data": {
    "tenant": {
      "id": "tnntten-DPXRgDGEaYBIffcfjp-Po",
      "name": "My Test Tenant",
      "description": "compose-stack walk through root tenant",
      "children": {
        "edges": [
          {
            "node": {
              "id": "tnntten-rXirlFQULBHDw9urtOjya",
              "name": "Development",
              "description": "Tenant for dev resources",
              "locations": {
                "edges": [
                  {
                    "node": {
                      "id": "lctnloc-sEdikPSFsjYJjJxssdTkY",
                      "name": "Just a test location",
                      "description": "location to show creating locations"
                    }
                  }
                ]
              }
            }
          }
        ]
      },
      "locations": {
        "edges": []
      }
    }
  }
}
```

### Traversing interfaces

Locations are owned by a ResourceOwner and not a Tenant. This requires the use of a GraphQL Fragment to traverse the graph from a Location to a Tenant. This is an example of a query that does that.

```
query {
  node(id: "lctnloc-sEdikPSFsjYJjJxssdTkY") {
    ... on Location {
      name
      owner {
        ... on Tenant {
          id
          name
          description
        }
      }
    }
  }
}
```

Which will return a response that includes the tenant name and description. This leverages the node-resolver service to determine the actual object type of the ResourceOwner id that is returned by location api.

```
{
  "data": {
    "node": {
      "name": "Just a test location",
      "owner": {
        "id": "tnntten-rXirlFQULBHDw9urtOjya",
        "name": "Development",
        "description": "Tenant for dev resources"
      }
    }
  }
}
```
