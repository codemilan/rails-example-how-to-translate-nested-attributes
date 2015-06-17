## How To Translate Nested Model Attributes in Error Messages

### Problem

I want to use localization to rename nested, polymorphic attributes
when used in errors for the parent model.

#### Example

* User model
* Address model, polymorphic
* Employer model

* User has one Mailing Address, polymorphic Address
* User has one Employer

* User validates `username`, `email`, `mailing_address`, and
`employer` for `presence`.
* Address validates `street`, `locality`, `region`, `country` and
`postal_code`.
* Employer validates `name`.

### Methodology

Setting new names for attributes in the config/locales/en.yml file as:

``` yaml
en:
  activerecord:
    attributes:
      address:
        locality: "City"
        region: "State"
      employer:
        name: "Nom"
      user:
        address.locality: "City"
        address.region: "State"
        mailing_address.locality: "City"
        mailing_address.region: "State"
        employer.name: "Societe Nom"
```

Building a new, unsaved user with new mailing address and new employer
(i.e., all fields are empty, which will not validate).

`u.errors.full_messages` will use the translations for the nested
employer, but **NOT** for the nested, polymorphic mailing address.

```
$ rails c
Loading development environment (Rails 4.2.1)
[1] pry(main)> user = User.new(mailing_address: Address.new, employer: Employer.new)
=> #<User:0x007f81e28ab4f0 id: nil, username: nil, lastname: nil, firstname: nil, email: nil, created_at: nil, updated_at: nil>
[2] pry(main)> user.valid?
=> false
[3] pry(main)> user.errors
=> #<ActiveModel::Errors:0x007f81e2951710
 @base=#<User:0x007f81e28ab4f0 id: nil, username: nil, lastname: nil, firstname: nil, email: nil, created_at: nil, updated_at: nil>,
 @messages=
  {:"employer.name"=>["can't be blank"],
   :"mailing_address.street"=>["can't be blank"],
   :"mailing_address.locality"=>["can't be blank"],
   :"mailing_address.region"=>["can't be blank"],
   :"mailing_address.country"=>["can't be blank"],
   :"mailing_address.postal_code"=>["can't be blank"],
   :username=>["can't be blank"],
   :email=>["can't be blank"]}>
[4] pry(main)> user.errors.full_messages
=> ["Nom can't be blank",
 "Mailing address street can't be blank",
 "Mailing address locality can't be blank",
 "Mailing address region can't be blank",
 "Mailing address country can't be blank",
 "Mailing address postal code can't be blank",
 "Username can't be blank",
 "Email can't be blank"]
```

You can see that `employer.name` in the errors collection was turned
into "Nom" in the full messages, while `mailing_address.locality` was
**NOT** turned into "City", nor `mailing_address.region` was **NOT**
turned into "State". <== *(remember what I said here, because it is wrong!)*

### Conclusion

What is the magic that will make a nested polymorphic table work like
a regular nested table?

Using the relationship names in the locale file:

``` yaml
en:
  activerecord:
    attributes:
      address:
        street: "Street"
        locality: "City"
        region: "State"
        country: "Country"
        postal_code: "Zip"
      mailing_address:
        street: "Street"
        locality: "City"
        region: "State"
        country: "Country"
        postal_code: "Zip"
      employer:
        name: "Nom"
      user:
        username: "Handle"
        email: "E-mail address"
```

This works!! Giving the entry `mailing_address` in the attributes list
tells Rails where to look for it.


```
$ rails c
Loading development environment (Rails 4.2.1)
[1] pry(main)> user = User.new(mailing_address: Address.new, employer: Employer.new)
=> #<User:0x007f9d519c8830 id: nil, username: nil, lastname: nil, firstname: nil, email: nil, created_at: nil, updated_at: nil>
[2] pry(main)> user.valid?
=> false
[3] pry(main)> user.errors
=> #<ActiveModel::Errors:0x007f9d4b6aacf8
 @base=#<User:0x007f9d519c8830 id: nil, username: nil, lastname: nil, firstname: nil, email: nil, created_at: nil, updated_at: nil>,
 @messages=
  {:"employer.name"=>["can't be blank"],
   :"mailing_address.street"=>["can't be blank"],
   :"mailing_address.locality"=>["can't be blank"],
   :"mailing_address.region"=>["can't be blank"],
   :"mailing_address.country"=>["can't be blank"],
   :"mailing_address.postal_code"=>["can't be blank"],
   :username=>["can't be blank"],
   :email=>["can't be blank"]}>
[4] pry(main)> user.errors.full_messages
=> ["Nom can't be blank",
 "Street can't be blank",
 "City can't be blank",
 "State can't be blank",
 "Country can't be blank",
 "Zip can't be blank",
 "Handle can't be blank",
 "E-mail address can't be blank"]
```

Note that what I *thought* was happening, that the yaml key somehow
had to be on the attribute itself was incorrect. I'm glad I used a
different translation in the first attempt, but I'm not so glad I
didn't see it sooner. :D

<del>The downside to this approach is that you end up repeating
yourself for every polymorphic relationship, as well as the
base.</del>

*Update:* And then, when I think about YAML, it's easy to DRY it up:

``` yaml
en:
  activerecord:
    attributes:
      address: &address
        street: "Street"
        locality: "City"
        region: "State"
        country: "Country"
        postal_code: "Zip"
      mailing_address:
        <<: *address
      employer:
        name: "Nom"
      user:
        username: "Handle"
        email: "E-mail address"
```

And upon even further thinking about how Rails' I18n tags work, it
makes perfect sense as the translation tags are keys joined by dots,
and I just should have realized what was going on in the beginning.


### Links Used

I read these, which proved to be useful, but in the end, the answer
revealed itself by experimentation and observation.

* <http://guides.rubyonrails.org/i18n.html#translations-for-active-record-models>
shows how to set up ActiveRecord attribute translations, and discusses
how to use them, but does not say anything about using them in
ActiveModel::Errors. For non-polymorphic error messages, they work
just fine. For the polymorphic error messages, however, they do not.

* <http://stackoverflow.com/questions/23714849/translation-for-nested-attributes-in-polymorphic-relationship>
also discusses how to use them in helpers and forms, but does not
touch on the error messages.
