# go-graphql-server-example

I've created an example of a GraphQL server using go-graphql. I hope it can be useful to someone who is learning GraphQL with Go. This example was tested using Go v1.12. This is source code from my blog post http://www.melvinvivas.com/develop-graphql-web-apis-using-golang

## Run the example
```
cd cd src/cmd/server/
go run main.go
```

## Access GraphiQL after running the example

http://localhost:3000/graphiql

Visit my blog http://www.melvinvivas.com for more tech articles

Related blog post http://www.melvinvivas.com/develop-graphql-web-apis-using-golang/


## Blog post

![How to create a GraphQL API Server using Go (Golang)](https://www.melvinvivas.com/content/images/2019/09/Screenshot-2019-09-18-15.36.37.png)

I've written an example on how to develop a [GraphQL API using Apollo Server](http://www.melvinvivas.com/graphql-api-using-apollo-server-example/). Using Apollo and NodeJS was quite straightforward. However, I wanted to use [Go(Golang)](https://golang.org/) for a new project I am working on. Initially, I hesitated in doing it in Go since it didn't look easy. After reading a few references and some perseverance, I managed to come up with an example on how to create a GraphQL using [graphql-go](https://github.com/graphql-go/graphql), n implementation of GraphQL for Go / Golang.

There are a few interesting frameworks you can start with when building your GraphQL APIs using Go. Here are a few from [GraphQL's website](https://graphql.org/code/#go):

- [graphql-go](https://github.com/graphql-go/graphql): An implementation of GraphQL for Go / Golang.
- [graph-gophers/graphql-go](https://github.com/graph-gophers/graphql-go): An active implementation of GraphQL in Golang (was [https://github.com/neelance/graphql-go](https://github.com/neelance/graphql-go)).
- [GQLGen](https://github.com/99designs/gqlgen) - Go generate based graphql server library.
- [graphql-relay-go](https://github.com/graphql-go/relay): A Go/Golang library to help construct a graphql-go server supporting react-relay.
- [machinebox/graphql](https://github.com/machinebox/graphql): An elegant low-level HTTP client for GraphQL.
- [samsarahq/thunder](https://github.com/samsarahq/thunder): A GraphQL implementation with easy schema building, live queries, and batching.

In this example, we'll be using [graphql-go](https://github.com/graphql-go/graphql), the first one in the list. Let's start.  Full source code is available in my github repo below.

{% github donvito/go-graphql-server-example %} 

Below is our main function. In the line 3, we are creating a GraphiQL Handler. This will enable GraphiQL for our api server so it doesn't not have to run separately.  [GraphiQL](https://github.com/graphql/graphiql) is an in-browser tool for exploring GraphQL APIs. Later on, once we're done with the code, we'll be able to see how GraphiQL looks like.  
  
Since we wanted our server to accept requests using http, we have created a server in line 10 running in port 3000 using the standard http package. We also need to handle GraphQL queries so we created a gqlHandler handler for it.


```
func main() {

  graphiqlHandler, err := graphiql.NewGraphiqlHandler("/graphql")
  if err != nil {
    panic(err)
  }

  http.Handle("/graphql", gqlHandler())
  http.Handle("/graphiql", graphiqlHandler)
  http.ListenAndServe(":3000", nil)

}
```


Below is the code for our handler. In this function, we receive the request and decode it to json. If there is an error in the parsing of the json, we return a HTTP error 400.

After that, the function processQuery() is called passing in the query.


```
func gqlHandler() http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if r.Body == nil {
      http.Error(w, "No query data", 400)
      return
    }

    var rBody reqBody
    err := json.NewDecoder(r.Body).Decode(&rBody)
    if err != nil {
      http.Error(w, "Error parsing JSON request body", 400)
    }

    fmt.Fprintf(w, "%s", processQuery(rBody.Query))

  })
}
```

Here how our process Query() function looks like. We retrieve the data from a json file for now using the dataFromJSON() function but ideally, this should be querying a database. After getting the data we'll be using, we call gqlSchema(jobsData) which creates the GraphQL schema from our data.  jobsData is an slice of Job structs and contains the data from our json file.


```
func processQuery(query string) (result string) {

  jobsData := dataFromJSON()

  params := graphql.Params{Schema: gqlSchema(jobsData), RequestString: query}
  r := graphql.Do(params)
  if len(r.Errors) > 0 {
    fmt.Printf("failed to execute graphql operation, errors: %+v", r.Errors)
  }
  rJSON, _ := json.Marshal(r)

  return fmt.Sprintf("%s", rJSON)

}

```

The GraphQL magic happens in the gqlSchema(jobsData) function. In this example, we have support for 2 queries, **/jobs** and **jobs/{id}**

**/jobs -** returns all jobs (line 3)  
**/jobs/{id} -** supports a single argument id and returns a single job filtered by id (line 10)  
  
The Resolvers for both functions are in line 6 and 18. Resolvers are responsible for returning the data for a query.

```
func gqlSchema(jobsData []Job) graphql.Schema {
  fields := graphql.Fields{
    "jobs": &graphql.Field{
      Type: graphql.NewList(jobType),
      Description: "All Jobs",
      Resolve: func(params graphql.ResolveParams) (interface{}, error) {
        return jobsData, nil
      },
    },
    "job": &graphql.Field{
      Type: jobType,
      Description: "Get Jobs by ID",
      Args: graphql.FieldConfigArgument{
        "id": &graphql.ArgumentConfig{
          Type: graphql.Int,
        },
      },
      Resolve: func(params graphql.ResolveParams) (interface{}, error) {
        id, success := params.Args["id"].(int)
        if success {
          for _, job := range jobsData {
            if int(job.ID) == id {
              return job, nil
            }
          }
        }
        return nil, nil
      },
    },
  }
  rootQuery := graphql.ObjectConfig{Name: "RootQuery", Fields: fields}
  schemaConfig := graphql.SchemaConfig{Query: graphql.NewObject(rootQuery)}
  schema, err := graphql.NewSchema(schemaConfig)
  if err != nil {
    fmt.Printf("failed to create new schema, error: %v", err)
  }

  return schema

}
```

## Run the GraphQL API server

```
git clone https://github.com/donvito/go-graphql-server-example.git
cd go-graphql-server-example/src/cmd/server/
go run main.go

```

## Access GraphiQL after running the server

[http://localhost:3000/graphiql](http://localhost:3000/graphiql)

![How to create a GraphQL API Server using Go (Golang)](https://www.melvinvivas.com/content/images/2019/09/Screenshot-2019-09-18-14.36.12-1.png)

That's it! We've now created a GraphQL API using Go.

Full source code is available in my github repo.  
[https://github.com/donvito/go-graphql-server-example](https://github.com/donvito/go-graphql-server-example)

**References**  
  
[https://github.com/graphql-go/graphql](https://github.com/graphql-go/graphql)  
[https://github.com/friendsofgo/graphiql](https://github.com/friendsofgo/graphiql)  
[https://graphql.org/code/#go](https://graphql.org/code/#go)

Cheers! Hope this helps anyone who is starting out with Go and GraphQL!

For more updates my new blog posts, you can follow me in Twitter [@donvito](https://twitter.com/donvito).  I also share code in my [GitHub](https://github.com/donvito). If you want to know more about what I do, please add me in [LinkedIn](https://www.linkedin.com/in/melvinvivas/). I recently started a new [youtube channel](https://www.youtube.com/channel/UCi6RVSV8s9Yy2Qg3WcGq9cg) - I upload some tutorials there as well. Check it out!

I'm also looking for remote development or tech blogging work. Would appreciate leads. :)  
[http://www.melvinvivas.com/looking-for-remote-software-development-work-outsource](http://www.melvinvivas.com/looking-for-remote-software-development-work-outsource/)


