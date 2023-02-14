---
title: golang - errors are easy, but at what cost?
categories: [programming]
tags: [golang, rust, errors]
---

## It's simple, right?

Go is [simple](https://go.dev/), and error handling should be equally simple. In other language such as Python or Ruby, you raise Exceptions if something goes wrong and catch them inside some sort of `try/except` block. In Go, if anything downstream of your function could error, you're expected to return two values: one value that _sometimes_ represents the object you're trying to return, and a second value that _sometimes_ represents the fact that something went wrong.

### Sometimes?

Let's look at an example of simple error handling in Go.

```go
func getUser(username string) (string, error) {
	// connect to some database and get user
	fullname, err := db.Find(username)
	if err != nil {
		return "", fmt.Errorf("getUser: failed to find user: %w", err)
	}
	return fullname, nil
}

func main() {
	userToFind := "lcrownover"
	fullname, err := getUser(userToFind)
	if err != nil {
		log.Fatalf("failed to find user %s: %w", userToFind, err)
        os.Exit(1)
	}
	fmt.Println("user's full name:", fullname)
}
```

Seems pretty straightforward, right? If the user is found, the `error` value is simply `nil`. If it can't find the user, you return... an empty string? Okay, I guess that makes sense. Go expects you to return the zero-value of whatever type you're returning, so an empty string it is. We'll just gloss over the fact that you're at the mercy of writing error checks every few lines. I wonder if career Go developers will eventually develop PTSD around having to constantly type
```go
if err != nil {
	return SomeZeroValue, fmt.Errorf("extra context: %w", err)
}
```
but that's just part of the job I guess.

So you deploy, and everything's working fine, until you need more than a username. What if you also need their full name? Maybe other pieces of information? So you decide to refactor and encapsulate this user data into a `User` struct, then pass that around.

```go
type User struct {
	Username string
	FullName string
	PartyPooper bool
}

func getUser(username string) (User, error) {
	// connect to some database and get user
	user := User{Username: username, PartyPooper: true}
	fullname, err := db.Find(username)
	if err != nil {
		return user, fmt.Errorf("getUser: failed to find user: %w", err)
	}
    user.FullName = fullname
	return user, nil
}

func main() {
	userToFind := "lcrownover"
	user, err := getUser(userToFind)
	if err != nil {
		log.Fatalf("failed to find user %s: %w", userToFind, err)
	}
	fmt.Printf("is %s fun at parties?: %v\n", user.FullName, user.PartyPooper)
}
```

Let's say you're searching for `Alyssa`, but she won the lottery last month and HR finally removed her account. The return values for `getUser` will be a mostly-filled-out `User` object and an `error`. The `User` could easily be mistaken for a valid user object, it's only missing one field. If you didn't `if err != nil` it, have fun tracking down where the hell this partially-provisioned struct came from.

If you Google for "golang return nil struct", hoping for a solution to this issue, you'll find that there is an alternative, returning nil pointers and relying on panics to uncover issues.

```go
type User struct {
	Name string
	Friends []*User
}

func getUser(username string) (*User, error) {
	name, err := db.Find(username)
	if err != nil {
		return nil, err
	}
	user := &User{
		Name:    name,
		Friends: []*User{},
	}
	return user, nil
}
```

If you return a pointer to `User`, you then have the option of returning `nil, err` if there's an error. This is quite a bit more straightforward, as you will never end up with partially-created structs. The issue then becomes, what happens later when I forget to write `if err != nil` and try to access `user.Name`? Panic!

If you're reading this looking for a clever solution, try looking elsewhere. Or go look at Rust's error handling, cause they have their shit figured out.
