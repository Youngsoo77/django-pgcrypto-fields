# django-pgcrypto-fields [![Build Status](https://travis-ci.org/incuna/django-pgcrypto-fields.svg?branch=master)](https://travis-ci.org/incuna/django-pgcrypto-fields?branch=master) [![Requirements Status](https://requires.io/github/incuna/django-pgcrypto-fields/requirements.svg?branch=master)](https://requires.io/github/incuna/django-pgcrypto-fields/requirements/?branch=master)

`django-pgcrypto-fields` is a `Django` extension which relies upon `pgcrypto` to
encrypt and decrypt data for fields.

`django-pgcrypto-fields` has 3 kinds of fields:
  - hash based fields
  - PGP fields
  - Symmetric fields

Hash based fields are:
 - `TextDigestField`
 - `TextHMACField`

PGP key fields are:
 - `EmailPGPPublicKeyField`
 - `IntegerPGPPublicKeyField`
 - `TextPGPPublicKeyField`

PGP symmetric fields are:
 - `EmailPGPSymmetricKeyField`
 - `IntegerPGPSymmetricKeyField`
 - `TextPGPSymmetricKeyField`


## Requirements

 - postgres with `pgcrypto`

## Installation

### pip

```bash
pip install django-pgcrypto-fields
```

### Fields

#### TextDigestField

`TextDigestField` is a hash based field. The value is hashed in the database when
saved with the `digest` pgcrypto function using the `sha512` algorithm.

#### TextHMACField

`TextHMACField` is a hash based field. The value is hashed in the database when
saved with the `hmac` pgcrypto function using a key and the `sha512` algorithm.

`key` is set in `settings.PGCRYPTO_KEY`.

#### EmailPGPPublicKeyField, IntegerPGPPublicKeyField and TextPGPPublicKeyField

Public key encryption. It generates a token generated with a public key to
encrypt the data and a private key to decrypt it.

Public and private keys can be set in settings with `PUBLIC_PGP_KEY` and
`PRIVATE_PGP_KEY`.

##### Generate GPG keys.

The public key is going to encrypt the message and the private key will be
needed to decrypt the content. The following commands have been taken from the
[pgcrypto documentation](http://www.postgresql.org/docs/devel/static/pgcrypto.html)
(see Generating PGP Keys with GnuPG).

Generating a public and a private key:

```bash
$ gpg --gen-key
$ gpg --list-secret-keys

/home/bob/.gnupg/secring.gpg
---------------------------
sec   2048R/21 2014-10-23
uid                  Test Key <example@example.com>
ssb   2048R/42 2014-10-23


$ gpg -a --export 42 > public.key
$ gpg -a --export-secret-keys 21 > private.key
```

#### EmailPGPSymmetricKeyField, IntegerPGPSymmetricKeyField and TextPGPSymmetricKeyField

Symmetric key encryption. Encrypt and decrypt the data with `settings.PGCRYPTO_KEY`.

### Django settings

In `settings.py`:
```python
BASEDIR = os.path.dirname(os.path.dirname(__file__))
PUBLIC_PGP_KEY_PATH = os.path.abspath(os.path.join(BASEDIR, 'public.key'))
PRIVATE_PGP_KEY_PATH = os.path.abspath(os.path.join(BASEDIR, 'private.key'))


# Used by PGPPublicKeyField
PUBLIC_PGP_KEY = open(PUBLIC_PGP_KEY_PATH).read()
PRIVATE_PGP_KEY = open(PRIVATE_PGP_KEY_PATH).read()

# Used by TextHMACField and PGPSymmetricKeyField
PGCRYPTO_KEY='ultrasecret'


# And add 'pgcrypto' to `INSTALLED_APPS` to create the extension for
# pgcrypto (in a migration).
INSTALLED_APPS = (
    ...
    'pgcrypto',
    ...
)

```

### Usage

#### Model definition

```python
from django.db import models

from pgcrypto import fields

class MyModel(models.Model):
    digest_field = fields.TextDigestField()
    hmac_field = fields.TextHMACField()

    integer_pgp_pub_field = fields.IntegerPGPPublicKeyField()
    pgp_pub_field = fields.TextPGPPublicKeyField()
    integer_pgp_sym_field = fields.IntegerPGPSymmetricKeyField()
    pgp_sym_field = fields.TextPGPSymmetricKeyField()
```

#### Encrypting

Data is encrypted when inserted into the database.

Example:
```python
>>> MyModel.objects.create(value='Value to be encrypted...')
```

#### Decrypting

##### PGP fields

When accessing the field name attribute on a model instance we are getting the
decrypted value.

Example:
```python
>>> # When using a PGP public key based encryption
>>> my_model = MyModel.objects.get()
>>> my_model.value  # field's proxy
'Value decrypted'
```

To be able to filter PGP values we first need to use an aggregate method to
decrypt the values.

Example when using a `PGPPublicKeyField`:
```python
>>> from pgcrypto.aggregates import PGPPublicKeyAggregate
>>> my_models = MyModel.objects.annotate(PGPPublicKeyAggregate('pgp_pub_field'))
[<MyModel: MyModel object>, <MyModel: MyModel object>]
>>> my_models.filter(pgp_pub_field__decrypted='Value decrypted')
[<MyModel: MyModel object>]
>>> my_models.first().pgp_pub_field__decrypted
'Value decrypted'
```

Example when using a `PGPSymmetricKeyField`:
```python
>>> from pgcrypto.aggregates import PGPSymmetricKeyAggregate
>>> my_models = MyModel.objects.annotate(PGPSymmetricKeyAggregate('pgp_sym_field'))
[<MyModel: MyModel object>, <MyModel: MyModel object>]
>>> my_models.filter(pgp_pub_field__decrypted='Value decrypted')
[<MyModel: MyModel object>]
>>> my_models.first().pgp_sym_field__decrypted
'Value decrypted'
```

##### Hash fields

To filter hash based values we need to compare hashes. This is achieved by using
a `__hash_of` lookup.

Example:
```python
>>> my_model = MyModel.objects.filter(digest_field__hash_of='value')
[<MyModel: MyModel object>]
>>> my_model = MyModel.objects.filter(hmac_field__hash_of='value')
[<MyModel: MyModel object>]

```
