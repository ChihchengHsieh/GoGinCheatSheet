# GoGinCheatSheet
For starting the go backend

## Create Directory in the go PATH

## Create the folder structure

- apis
- databases
- middlewares
- models
- utils
- routers

## Create main.go

```go
package main

func main(){}
```

## Create the .env file for loading the env variables

```
MGDB_APIKEY=
JWT_SECRET=
```

#### Adding autoload to main.go

```go
import (
	_ "github.com/joho/godotenv/autoload"
)
```


## Adding Database

#### In the databases/mongodb.go

```go

package databases

import (
	"context"
	"fmt"
	"log"
	"os"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// DB - MongoDB database
var DB *mongo.Database

// InitDB - Initialise the database for MongoDB
func InitDB() {
	// Setting autoload in the main funciton to get the environment variables
	clientOptions := options.Client().ApplyURI(os.Getenv("MGDB_APIKEY"))
	client, err := mongo.Connect(context.TODO(), clientOptions)

	if err != nil {
		log.Fatal(err)
	}

	// Checking
	err = client.Ping(context.TODO(), nil)

	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("Connected to MongoDB successfully.")

	DB = client.Database("DB_NAME")
}
```


## Deciding Schema and adding Models

#### In models/user.go


```go
package models

import (
	"go.mongodb.org/mongo-driver/bson/primitive"
)

// User - Schema for User
type User struct {
	ID       primitive.ObjectID `json:"_id" bson:"_id, omitempty"`
	Email    string             `json:"email" bson:"email"`
	Password string             `json:"password" bson:"password"`
}

```


## Adding Manipulating Functions


```go


// AddUser - Add a user to the database
func AddUser(inputUser *User) (interface{}, error) {
	result, err := databases.DB.Collection("user").InsertOne(context.TODO(), inputUser)
	return result.InsertedID, err
}

```

## Adding Api functions


```go
func SignupUser(c *gin.Context) {
	// Extract required fields, including "email", "password" and "code"
	email, password, code := c.PostForm("email"), c.PostForm("password"), c.PostForm("code")

	// Checking if the password or password is empty
	if email == "" || password == "" {
		c.JSON(http.StatusBadRequest, gin.H{
			"error":    "You must provide email and password",
			"msg":      "You must provide email and password",
			"email":    email,
			"password": password,
		})
		return
	}

	// Checking if the register code is correct
	if code != os.Getenv("REGISTER_CODE") && code != os.Getenv("ADMIN_REGISTER_CODE") {
		c.JSON(http.StatusBadRequest, gin.H{
			"err":  "The Given Rigister Code is not correct",
			"msg":  "The Given Rigister Code is not correct",
			"code": code,
		})
		return
	}

	// Checking if the user already exist
	if user, err := models.FindUserByEmail(email); user != nil {
		c.JSON(http.StatusConflict, gin.H{
			"err":  err,
			"msg":  "The user already exists.",
			"user": user,
		})
		return
	}

	// Using the register code to define the role
	role := "normal"
	if code == os.Getenv("ADMIN_REGISTER_CODE") {
		role = "admin"
	}

	// Hash the given password
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"err": err,
			"msg": "Cannot hash the given password",
		})
		return
	}

	// Create the newUser
	newUser := models.User{
		Email:    email,
		Password: string(hashedPassword),
		Role:     role,
	}

	// Add this User
	insertedID, err := models.AddUser(&newUser)

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"error":   err,
			"msg":     "Cannot register this user",
			"newUser": newUser,
		})
		return
	}

	newUser.ID = insertedID.(primitive.ObjectID)

	newUser.Password = ""

	authToken, err := utils.GenerateAuthToken(newUser.ID.Hex())

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"error": err,
			"msg":   "Cannot generate the auth token for this user",
			"user":  newUser,
		})
	}

	c.JSON(http.StatusOK, gin.H{
		"user":  newUser,
		"token": authToken,
	})

}
```


## Adding Router Setup


#### routers/index.go


```go
package routers

import (
	"github.com/gin-gonic/gin"
)

// InitRouter - Initialise all the router in this function
func InitRouter() *gin.Engine {
	router := gin.Default()
	UserRouterInit(router)
	return router
}
```

#### routers/user.go

```go
package routers

import (
	"blog/apis"
	"blog/middlewares"

	"github.com/gin-gonic/gin"
)

// UserRouterInit - Initialise userRouter
func UserRouterInit(router *gin.Engine) {
	userRouter := router.Group("/user")
	userRouter.Use(middlewares.LoginAuth()) // Why just return a function?
	{
		userRouter.POST("/signup", apis.SignupUser)
		userRouter.POST("/login", apis.LoginUser)
	}
}

```


## Prepaire the Main.go

```go
package main

import (
	"blog/databases"
	"blog/routers"

	"github.com/gin-gonic/gin"
	_ "github.com/joho/godotenv/autoload"
)

func main() {
	gin.ForceConsoleColor()
	databases.InitDB()
	router := routers.InitRouter()
	router.Run()
}

```

