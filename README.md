**Why is GraphQL gaining adoption?**

**The problem**  
As a senior developer or a system architect, in the era of enterprise products and microservice architecture, you probably had the chance to develop hundreds of API endpoints. The most common approach to implementing these endpoints is to use the RESTful API design. You can handle all your CRUD operations by creating multiple endpoints for each resource. However, as the product grows bigger and the number of endpoints increases, you may face some of the following challenges:

* Managing and maintaining multiple endpoints can be a painful experience  
* Returning the whole payload once a request is received (aka ‘over-fetching’ data)  
* Returning too little data ( aka ‘under-fetching’ data)  
* Having to fetch data from multiple sources

Some or all of the items mentioned above can become a bottle neck and may affect the system performance.   
   
Before we continue, let me show you an example. We have a simple list of book authors entity in our product. Each item in the list has an id, a name and a nested list books containing book titles:

```json
{  
  "data": {  
    "authors": [  
      {  
        "id": 1,  
        "name": "George Orwell",  
        "books": [  
          {"title": "1984"},  
          {"title": "Animal Farm"}  
        ]  
      },  
      {  
        "id": 2,  
        "name": "J.K. Rowling",  
        "books": [  
          {"title": "Harry Potter and the Philosopher's Stone"}  
        ]  
      }  
    ]  
  }  
}
```

We may want to have the following endpoints to implement CRUD operations:

| HTTP Method | Endpoint | Purpose |
| ----- | ----- | ----- |
| **GET** | `/authors` | Get all authors |
| **GET** | `/authors/{id}` | Get one author by ID |
| **POST** | `/authors` | Create a new author |
| **PUT** | `/authors/{id}` | Update an author (replace) |
| **PATCH** | `/authors/{id}` | Update part of an author (partial update) |
| **DELETE** | `/authors/{id}` | Delete an author |

**The solution**  
Improving system performance is broad topic and there are many factors that can affect the overall performance of a system. Here in this article, we are mostly focusing on code optimization and going to talk about a solution that you can implement, as a senior developer. An API design approach that has recently gained popularity. It is called GraphQL design. 

Like RESTful, it is an API architecture. It works over http and can return JSON. 

Unlike RESTful, there is no need to implement several endpoints for each of your resources, and all your CRUD operations for a single resource can be handled using only one URL. As the name says, like SQL query, it is a query language that enables you to filter the API response using queries instead of receiving the whole payload. 

![Image](https://github.com/user-attachments/assets/1778b925-2f0b-4435-8107-0895ef1c9c19)


In our sample code, the only endpoint you require to implement all CRUD operations is ‘**/graphql’.** Depending on the input object and parameters in the request object, the result would act differently.

Here are some request and response examples:

| Request | Response | Purpose |
| :---- | :---- | ----- |
| {   authors {     id     name     books {       title     }   } } | {   "data": {     "authors": [       {         "id": 1,         "name": "George Orwell",         "books": [           {"title": "1984"},           {"title": "Animal Farm"}         ]       },       {         "id": 2,         "name": "J.K. Rowling",         "books": [           {"title": "Harry Potter and the Philosopher's Stone"}         ]       }     ]   } }  | Get all authors with their books |
| {   books {     title   } }  | {   "data": {     "books": [       {         "title": "1984"       },       {         "title": "Animal Farm"       },       {         "title": "Harry Potter and the Philosopher's Stone"       }     ]   } }  | Just get book titles  |
| {   author(id: 1\) {     name     books {       title     }   } }  | {   "data": {     "author": {       "name": "George Orwell",       "books": [         {"title": "1984"},         {"title": "Animal Farm"}       ]     }   } }  | Get a specific author and the books |

As you can see in the table above, you have absolute control over what the endpoint responds with by changing the request object. 

**Sample Code**  
Below is a Python sample code of implementing both RESTful and GraphQL using FastAPI. As mentioned above, there are multiple endpoints in RESTful design and only one in GraphQL.

```python
# app.py  
from fastapi import FastAPI, HTTPException  
from pydantic import BaseModel  
from typing import List, Optional  
import strawberry  
from strawberry.fastapi import GraphQLRouter

# -------------------------  
# Data models (in-memory)  
# -------------------------  
class Author(BaseModel):  
   id: int  
   name: str

class Book(BaseModel):  
   id: int  
   title: str  
   author_id: int

authors = [  
   Author(id=1, name="George Orwell"),  
   Author(id=2, name="J.K. Rowling"),  
]

books = [  
   Book(id=1, title="1984", author_id=1),  
   Book(id=2, title="Animal Farm", author_id=1),  
   Book(id=3, title="Harry Potter and the Philosopher's Stone", author_id=2),  
]

# -------------------------  
# GraphQL Schema  
# -------------------------  
@strawberry.type  
class BookType:  
   id: int  
   title: str  
   author_id: int

   @strawberry.field  
   def author(self) -> "AuthorType":  
       for author in authors:  
           if author.id == self.author_id:  
               return author  
       return None

@strawberry.type  
class AuthorType:  
   id: int  
   name: str

   @strawberry.field  
   def books(self) -> List[BookType]:  
       return [book for book in books if book.author_id == self.id]

# -------------------------  
# Query resolvers  
# -------------------------  
def get_authors() -> List[AuthorType]:  
   return authors

def get_books() -> List[BookType]:  
   return books

def get_author(id: int) -> Optional[AuthorType]:  
   return next((a for a in authors if a.id == id), None)

def get_book(id: int) -> Optional[BookType]:  
   return next((b for b in books if b.id == id), None)

def get_book_by_title(title: str) -> Optional[BookType]:  
   return next((b for b in books if b.title.lower() == title.lower()), None)

@strawberry.type  
class Query:  
   authors: List[AuthorType] = strawberry.field(resolver=get_authors)  
   books: List[BookType] = strawberry.field(resolver=get_books)  
   author: Optional[AuthorType] = strawberry.field(resolver=get_author)  
   book: Optional[BookType] = strawberry.field(resolver=get_book)  
   bookByTitle: Optional[BookType] = strawberry.field(resolver=get_book_by_title)

# -------------------------  
# Mutations  
# -------------------------  
@strawberry.type  
class Mutation:  
   @strawberry.mutation  
   def addBook(self, title: str, author_id: int) -> BookType:  
       new_id = max(book.id for book in books) + 1 if books else 1  
       new_book = Book(id=new_id, title=title, author_id=author_id)  
       books.append(new_book)  
       return new_book

schema = strawberry.Schema(query=Query, mutation=Mutation)

# -------------------------  
# FastAPI app  
# -------------------------  
app = FastAPI(title="Books API (REST + GraphQL)")  
graphql_app = GraphQLRouter(schema)  
app.include_router(graphql_app, prefix="/graphql")

# -------------------------  
# REST Endpoints  
# -------------------------  
@app.get("/authors", response_model=List[Author])  
def get_authors_rest():  
   return authors   

@app.get("/authors/{author_id}", response_model=Author)  
def get_author_rest(author_id: int):  
   author = next((a for a in authors if a.id == author_id), None)  
   if not author:  
       raise HTTPException(status_code=404, detail="Author not found")  
   return author

@app.get("/books", response_model=List[Book])  
def get_books_rest():  
   return books

@app.get("/books/{book_id}", response_model=Book)  
def get_book_rest(book_id: int):  
   book = next((b for b in books if b.id == book_id), None)  
   if not book:  
       raise HTTPException(status_code=404, detail="Book not found")  
   return book

@app.post("/books", response_model=Book)  # REST mutation equivalent  
def create_book(book: Book):  
   books.append(book)  
   return book
```

**Benefits and limitations**  
Software development is about tradeoffs. There is no one right or wrong answer. Choosing between RESTful and GraphQL is no exception, and you, as a system architect or a decision maker, need to consider all the possibilities and challenges when deciding to switch to GraphQL. Here is a side-by-side example of the difference between the two:

| Aspect | REST | GraphQL |
| ----- | ----- | ----- |
| **Endpoints** | Multiple (`/users`, `/users/:id/posts`, etc.) | Single (`/graphql`) |
| **Data Fetching** | Risk of **over-fetching** (too much data) or **under-fetching** (too little data) | You can specifie exactly what fields are needed |
| **Requests Needed** | Multiple requests for related data (user > posts > comments) | One request can return nested/related data |
| **Performance** | May require multiple calls (not suitable for mobile apps) | Efficient: one round-trip with exactly needed data |
| **Use Cases** | Simple CRUD, file uploads | Complex data, mobile apps, dashboards, when minimizing requests matters |

 You can start using GraphQL if your UI requires complex data, if you want to minimize the number of API calls for an action or if you want strongly typed schemas and self-documenting APIs.
