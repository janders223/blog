+++
title = "The Craziest Terraform for_each I've Ever Written"
author = ["Jim Anders"]
publishDate = 2020-06-10T00:00:00-04:00
tags = ["terraform", "dry"]
draft = false
+++

Today I set out on a mission to refactor some code that has been bothering me for some time. The verbosity of terraform code often often bothers me and I _always_ look for opportunities to [DRY](https://en.wikipedia.org/wiki/Don%27t%5Frepeat%5Fyourself) up my code. The particular code of interest here is deploying multiple instances of virtual machines into multiple resource groups in Azure.

The first iteration looked something similar to the following.

```terraform
resource "azurerm_linux_virtual_machine" "myvm" {
  count               = var.number_of_instances
  name                = "myvm-${count.index}"
  resource_group_name = data.azurerm_resource_group.resource_group1.name
  location            = data.azurerm_resource_group.resource_group1.location
  ...
}

resource "azurerm_linux_virtual_machine" "myvm" {
  count               = var.number_of_instances
  name                = "myvm-${count.index}"
  resource_group_name = data.azurerm_resource_group.resource_group2.name
  location            = data.azurerm_resource_group.resource_group2.location
  ...
}
```

And this worked, but it's painful. Painful to update, painful to look at and painful to maintain. The first part of this refactor is to get something to iterate over.

```terraform
locals {
  resource_groups = {
    rg1 = data.azurerm_resource_group.resource_group1
    rg2 = data.azurerm_resource_group.resource_group2
  }
}
```

And of course iterating over creates a single vm in each location.

```terraform
resource "azurerm_linux_virtual_machine" "myvm" {
  for_each            = local.resource_groups
  name                = "myvm-${each.key}"
  resource_group_name = each.value.name
  location            = each.value.location
  ...
}
```

And this is great, but it only create 2 servers. Of course, you cannot use `for_each` and `count` together in the same resource. My first thought was to extract the resource to a module and use `for_each` on the module passing a `count` variable. This also does not work as `for_each` and `count` are reserved for use only on resources and will not work with modules.

So what I needed was a data structure that I could iterate over with `for_each` that took into account the number of instances that I needed and the number of resource group locations that I have. To get the total number of instances I needed something that multiplied the number of locations by the number of instances asked for.

```terraform
locals {
  resource_group_keys = keys(local.resource_groups)
  number_locations    = length(local.resource_group_keys)
  total_instances     = local.number_locations * var.number_of_instances
}
```

Fortunately, terraform has many built in functions to aid in building up the data structure. [`keys`](https://www.terraform.io/docs/configuration/functions/keys.html) iterates over a Map and returns a List of all keys which is then passed to [`length`](https://www.terraform.io/docs/configuration/functions/length.html) to get a count of 2 in this case. The builtin multiplication function is then used to get a total number of instances. So if `number_of_instances` is set to 4 it will build 8 total instances, 4 in each resource group.

Now building up the data structure to iterate over is a little more involved.

```terraform
resource "azurerm_linux_virtual_machine" "my-vm" {
  for_each = {
    for num in range(local.total_instances) : tostring(num) => {
      lookup                  = local.resource_group_keys[(num % local.number_locations)]
      resource_group_location = local.resource_groups[local.resource_group_keys[(num % local.number_locations)]].name
      resource_group_name     = local.resource_groups[local.resource_group_keys[(num % local.number_locations)]].location
      name                    = format("my-vm-%s-%s", local.resource_group_keys[(num % local.number_locations)], num > local.number_instances ? ((num + 1) / local.number_locations) : num)
    }
  }
```

There is a lot going on here so let's unpack it.

```terraform
    for num in range(local.total_instances) : tostring(num) => {}
```

The `for x in List : x => {}` allows iteration over the list returning a new Map with a `x` as the key and the empty Map as the value. The [`range`](https://www.terraform.io/docs/configuration/functions/range.html) function return a List expanding the numbers by the provided value. Going with 4 instances, this returns `[0, 1, 2, 3, 4, 5, 6, 7]` which is collected into a new Map with the stringified number as the key.

```terraform
{
  "0" = {}
  "1" = {}
  "2" = {}
  "3" = {}
  "4" = {}
  "5" = {}
  "6" = {}
  "7" = {}
}
```

Looking at the vm resource from the Azure Provider, the information that is needed to iterate over is the resource group name, resource group location and each vm needs a unique name. A value is also needed to lookup any resources that were previously created by using `for_each` with the `resource_groups` local. So building out each part of that data structure into the Map goes like this.

```terraform
      lookup = local.resource_group_keys[(num % local.number_locations)]
```

`local.resource_group_keys` is a List of the keys realting to the resource group data objects. In this case it's value is ~["rg1", "rg2"]. Lists are like normal Arrays in that they are zero indexed and values are retrived by index. Given 2 values, indicies are 0 and 1, and with a little [modulo math](https://en.wikipedia.org/wiki/Modulo%5Foperation) with return a 0 or a 1.

```terraform
0 % 2 => 0
1 % 2 => 1
4 % 2 => 0
```

So after this line, the data structure looks like this.

```terraform
{
  "0" = {
    lookup = "rg1"
  }
  "1" = {
    lookup = "rg2"
  }
  "2" = {
    lookup = "rg1"
  }
  "3" = {
    lookup = "rg2"
  }
  "4" = {
    lookup = "rg1"
  }
  "5" = {
    lookup = "rg2"
  }
  "6" = {
    lookup = "rg1"
  }
  "7" = {
    lookup = "rg2"
  }
}
```

The next two lines set the information for the resource group using the exact same logic of List indicies, modulo math and getting a Map value by key. `(num % local.number_locations)` returns a 1 or a 0 to grab the correct key from the List `local.resource_group_keys` which is then used to the data object from the Map `local.resource_groups` by key.

```terraform
resource_group_location = local.resource_groups[local.resource_group_keys[(num % local.number_locations)]].name
resource_group_name     = local.resource_groups[local.resource_group_keys[(num % local.number_locations)]].location
```

The data structure now looks as follows.

```terraform
{
  "0" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
  }
  "1" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
  }
  "2" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
  }
  "3" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
  }
  "4" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
  }
  "5" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
  }
  "6" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
  }
  "7" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
  }
}
```

Last up is the name key which is used for generating a unique resource name which is also used to set the vm hostname. The [`format`](https://www.terraform.io/docs/configuration/functions/format.html) function has the string to be formatted with two values. The first is the lookup key accessed in the same way as above since `self` is not supported here. Not wanting hostnames to have 0, 2, 4, 6 in one resource group and 1, 3, 5, 7 in the other, a ternary is used to check when the loop gets over the number of required instances, dividing it by the number of locations to get back the correct values.

```terraform
name = format("my-vm-%s-%s", local.resource_group_keys[(num % local.number_locations)], num >= (local.number_instances - 1) ? ((num + 1) / local.number_locations) : num)
```

The final data structure is.

```terraform
{
  "0" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
    name                    = "mv-vm-eastus2-0"
  }
  "1" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
    name                    = "mv-vm-centralus-0"
  }
  "2" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
    name                    = "mv-vm-eastus2-1"
  }
  "3" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
    name                    = "mv-vm-centralus-1"
  }
  "4" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
    name                    = "mv-vm-eastus2-2"
  }
  "5" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
    name                    = "mv-vm-centralus-2"
  }
  "6" = {
    lookup                  = "rg1"
    resource_group_location = "eastus2"
    resource_group_name     = "rg1"
    name                    = "mv-vm-eastus2-3"
  }
  "7" = {
    lookup                  = "rg2"
    resource_group_location = "centralus"
    resource_group_name     = "rg2"
    name                    = "mv-vm-centralus-3"
  }
}
```

Looping through this now provides the required number of the vm reource in each resource group.

```terraform
resource "azurerm_linux_virtual_machine" "my-vm" {
  for_each = {...}
  resource_group_location = each.value.resource_group_location
  resource_group_name     = each.value.resource_group_name
  name                    = each.value.name
}
```
