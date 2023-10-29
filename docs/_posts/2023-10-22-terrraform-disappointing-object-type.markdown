---
layout: post
title:  "Terraform's disappointing object type"
date:   2023-10-22 01:37:24 +0100
categories: terraform
---

In go, the following code does not compile

```
package main

import "fmt"

type cloudResource struct {
	name string
	sku  string
}

func main() {
	config := cloudResource{name: "myresource", sku: "prod", location: "us"}
	fmt.Println(config)
}

```

This is the error I get

```
unknown field location in struct literal of type cloudResource
```

Essentially, I can't just add a field to a struct at calling time. If I need a field called location, the struct needs to have it as part of its definition.

Unfortunately, terraform does not work like this and the object type can only be used to specify the fields that the object **must** have rather than a requirement to match the object definition field by field. In other words, there is nothing to prevent you from adding extra fields at calling time, e.g. in your tfvars file.

This, incidentally, is by design. From Hashicorp's documentation for terraform [Type Constraints](https://developer.hashicorp.com/terraform/language/expressions/type-constraints#structural-types):

| Values that match the object type must contain all of the specified keys, and the value for each key must match its specified type. (Values with additional keys can still match an object type, but the extra attributes are discarded during type conversion.)

Take this code:

```
variable "mytype" {
    description = "An object"
    type = object({
      name = string
      sku = optional(string, "dev")
    })  
    default = {
      name = "value"
      sku = "othervalue"
      location = "a location"
    }
}

output "myoutput" {
    value =var.mytype
}
```
The code above returns this after terraform apply:

```
Outputs:

myoutput = {
  "name" = "value"
  "sku" = "othervalue"
}
```

¯\\\_(ツ)\_/¯
