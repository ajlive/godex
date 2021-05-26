# godex

A pattern for validating data with generics based on [pydantic](https://github.com/samuelcolvin/pydantic) ([docs](https://pydantic-docs.helpmanual.io/))

This is, uh, _very_ early stages. So far it's just 83 lines of primitive, but working, code. But I have already written a PHP version of substantial parts of pydantic, so I'm goint to claim I know what I'm doing and that this is all going to go just fine.

It may turn out to be an experiment demonstrating why you shouldn't do pydantic in Go. It may turn out to be an interesting and even useful pattern.

🤷

go2go playground link: <https://go2goplay.golang.org/p/dyx4Ui_ldBc>

```golang
package main

import (
	"fmt"
	"os"
)

type Contract[T any] struct {
	validators []func(T) (T, string, error)
}

func (c *Contract[T]) validator(vdtr func(T) (T, string, error)) {
	c.validators = append(c.validators, vdtr)
}

func (c *Contract[T]) validData(data T) (validData T, errors map[string][]error) {
	var field string
	var err error
	errors = make(map[string][]error)
	for _, vdtr := range c.validators {
		validData, field, err = vdtr(data)
		if err == nil {
			continue
		}
		if _, ok := errors[field]; !ok {
			errors[field] = make([]error, 0)
		}
		errors[field] = append(errors[field], err)
	}
	return validData, errors
}

type Person struct {
	id   int
	name string
}

func main() {
	// declare contract
	c := Contract[Person]{
		validators: make([]func(Person) (Person, string, error), 0),
	}
	c.validator(func(p Person) (Person, string, error) {
		field := "id"
		if p.id <= 0 {
			return p, field, fmt.Errorf("id must be positive integer; got %v", p.id)
		}
		return p, field, nil
	})
	c.validator(func(p Person) (Person, string, error) {
		field := "name"
		if p.name == "" {
			return p, field, fmt.Errorf("no name for person with id %v", p.id)
		}
		return p, field, nil
	})

	var errors map[string][]error

	// validate valid data
	data := Person{
		id:   1,
		name: "Guy Incognito",
	}
	validData, errors := c.validData(data)
	if len(errors) != 0 {
		fmt.Printf("invalid Person: %+v\nerrors: %+v", validData, errors)
		os.Exit(1)
	}
	fmt.Printf("valid Person: %+v\n", validData)

	fmt.Println("")

	// invalidate bad data
	badData := Person{}
	invalidData, errors := c.validData(badData)
	if len(errors) != 0 {
		fmt.Printf("invalid Person: %+v\nerrors: %+v", invalidData, errors)
		os.Exit(1)
	}
	fmt.Printf("valid Person: %+v\n", invalidData)

}
```

Output:
```
valid Person: {id:1 name:Guy Incognito}

invalid Person: {id:0 name:}
errors: map[id:[id must be positive integer; got 0] name:[no name for person with id 0]]
Program exited: status 1.
```
