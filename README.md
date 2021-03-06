# NAME

DBIx::Class::Sims - The addition of simulating data to DBIx::Class

# SYNOPSIS

Within your schema class:

    __PACKAGE__->load_components('Sims');

Within your resultsources, specify the sims generation rules for columns that
need specified.

    __PACKAGE__->add_columns(
      ...
      address => {
        data_type => 'varchar',
        is_nullable => 1,
        data_length => 10,
        sim => { type => 'us_address' },
      },
      zipcode => {
        data_type => 'varchar',
        is_nullable => 1,
        data_length => 10,
        sim => { type => 'us_zipcode' },
      },
      column1 => {
        data_type => 'int',
        is_nullable => 0,
        sim => {
          min => 10,
          max => 20,
        },
      },
      column2 => {
        data_type => 'varchar',
        is_nullable => 1,
        data_length => 10,
        default_value => 'foobar',
      },
      ...
    );

Later:

    $schema->deploy({
      add_drop_table => 1,
    });

    my $ids = $schema->load_sims({
      Table1 => [
        {}, # Take sims or default values for everything
        { # Override some values, take sim values for others
          column1 => 20,
          column2 => 'something',
        },
      ],
    });

# PURPOSE

Generating test data for non-simplistic databases is extremely hard, especially
as the schema grows and changes. Designing scenarios __should__ be doable by only
specifying the minimal elements actually used in the test with the test being
resilient to any changes in the schema that don't affect the elements specified.
This includes changes like adding a new parent table, new required child tables,
and new non-NULL columns to the table being tested.

With Sims, you specify only what you care about. Any required parent rows are
automatically generated. If a row requires a certain number of child rows (all
artists must have one or more albums), that can be set as well. If a column must
have specific data in it (a US zipcode or a range of numbers), you can specify
that in the table definition.

And, in all cases, you can override anything.

# DESCRIPTION

This is a [DBIx::Class](http://search.cpan.org/perldoc?DBIx::Class) component that adds a few methods to your
[DBIx::Class::Schema](http://search.cpan.org/perldoc?DBIx::Class::Schema) object. These methods make it much easier to create data
for testing purposes (though, obviously, it's not limited to just test data).

# METHODS

## load\_sims

`$rv = $schema->load_sims( $spec, ?$constraints, ?$hooks )`

This method will load the rows requested in `$spec`, plus any additional rows
necessary to make those rows work. This includes any parent rows (as defined by
`belongs_to`) and per any constraints defined in `$constraints`. If need-be,
you can pass in hooks (as described below) to manipulate the data.

load\_sims does all of its work within a call to ["txn\_do" in DBIx::Class::Schema](http://search.cpan.org/perldoc?DBIx::Class::Schema#txn\_do).
If anything goes wrong, load\_sims will rethrow the error after the transaction
is rolled back.

This, of course, assumes that the tables you are working with support
transactions. (I'm looking at you, MyISAM!) If they do not, that is on you.

### Return value

This will return a hash of arrays of hashes. This will match the `$spec`,
except that where the `$spec` has a requested set of things to make, the return
will have the primary columns.

Examples:

If you have a table foo with "id" as the primary column and you requested:

    {
      Foo => [
        { name => 'bar' },
      ],
    }

You will receive back (assuming the next id value is 1):

    {
      Foo => [
        { id => 1 },
      ],
    }

If you have a table foo with "name" and "type" as the primary columns and you
requested:

    {
      Foo => [
        { children => [ {} ] },
      ],
    }

You will receive back (assuming the next PK values are as below):

    {
      Foo => [
        { name => 'bar', type => 'blah' },
      ],
    }

Note that you do not get back the ids for any additional rows generated (such as
for the children). 

## set\_sim\_type

`$class_or_obj->set_sim_type({ $name => $handler, ... });`

This method will set the handler for the `$name` sim type. The handler must be
a reference to a subroutine. You may pass in as many name/handler pairs as you
like.

This method may be called as a class or object method.

This method returns nothing.

`set_sim_types()` is an alias to this method.

# SPECIFICATION

The specification can be passed along as a filename that contains YAML or JSON,
a string that contains YAML or JSON, or as a hash of arrays of hashes. The
structure should look like:

    {
      ResultSourceName => [
        {
          column => $value,
          column => $value,
          relationship => {
            column => $value,
          },
          'relationship.column' => $value,
          'rel1.rel2.rel3.column' => $value,
        },
      ],
    }

If a column is a belongs\_to relationship name, then the row associated with that
relationship specifier will be used. This is how you would specify a specific
parent-child relationship. (Otherwise, a random choice will be made as to which
parent to use, creating one as necessary if possible.) The dots will be followed
as far as necessary.

Columns that have not been specified will be populated in one of two ways. The
first is if the database has a default value for it. Otherwise, you can specify
the `sim` key in the column\_info for that column. This is a new key that is not
used by any other component. See ["SIM ENTRY"](#SIM ENTRY) for more information.

(Please see ["add\_columns" in DBIx::Class::ResultSource](http://search.cpan.org/perldoc?DBIx::Class::ResultSource#add\_columns) for details on column\_info)

__NOTE__: The keys of the outermost hash are resultsource names. The keys within
the row-specific hashes are either columns or relationships. Not resultsources.

# CONSTRAINTS

The constraints can be passed along as a filename that contains YAML or JSON, a
string that contains YAML or JSON, or as a hash of arrays of hashes. The
structure should look like:

    {
      Person => {
        addresses => 2,
      },
    }

All the `belongs_to` relationships are automatically added to the constraints.
You can add additional constraints, as needed. The most common use for this will
be to add required child rows. For example, `Person->has_many('addresses')`
would normally mean that if you create a Person, no Address rows would be
created.  But, we could specify a constraint that says "Every person must have
at least 2 addresses." Now, whenever a Person is created, two Addresses will be
added along as well, if they weren't already created through some other
specification.

# HOOKS

Most people will never need to use this. But, some schema definitions may have
reasons that prevent a clean simulating with this module. For example, there may
be application-managed sequences. To that end, you may specify the following
hooks:

- preprocess

    This receives `$name, $source, $spec` and expects nothing in return. `$spec`
    is the hashref that will be passed to `<$schema-`resultset($name)->create()>>.
    This hook is expected to modify `$spec` as needed.

- postprocess

    This receives `$name, $source, $row` and expects nothing in return. This hook
    is expected to modify the newly-created row object as needed.

# SIM ENTRY

To control how a column's values are simulated, add a "sim" entry in the
column\_info for that column. The sim entry is a hash that can have the followingkeys:

- value

    This behaves just like default\_value would behave, but doesn't require setting a
    default value on the column.

        sim => {
            value => 'The value to always use',
        },

- type

    This labels the column as having a certain type. A type is registered using
    ["set\_sim\_type"](#set\_sim\_type). The type acts as a name for a function that's used to generate
    the value. See ["Types"](#Types) for more information.

- min / max

    If the column is numeric, then the min and max bound the random value generated.
    If the column is a string, then the min and max are the length of the random
    value generated.

- func

    This is a function that is provided the column info. Its return value is used to
    populate the column.

- null\_chance

    If the column is nullable _and_ this is set _and_ it is a number between 0 and
    1, then if `rand()` is less than that number, the column will be set to null.
    Otherwise, the standard behaviors will apply.

    If the column is __not__ nullable, this setting is ignored.

(Please see ["add\_columns" in DBIx::Class::ResultSource](http://search.cpan.org/perldoc?DBIx::Class::ResultSource#add\_columns) for details on column\_info)

## Types

The handler for a sim type will receive the column info (as defined in
["add\_columns" in DBIx::Class::ResultSource](http://search.cpan.org/perldoc?DBIx::Class::ResultSource#add\_columns)). From that, the handler returns the
value that will be used for this column.

Please see [DBIx::Class::Sims::Types](http://search.cpan.org/perldoc?DBIx::Class::Sims::Types) for the list of included sim types.

# TODO

## Multi-column types

In some applications, columns like "state" and "zipcode" are correlated. Values
for one must be legal for the value in the other. The Sims currently has no way
of generating correlated columns like this.

This is most useful for saying "These 6 columns should be a coherent address".

## Allow a column to reference other columns

Sometimes, a column should alter its behavior based on other columns. A fullname
column may have the firstname and lastname columns concatenated, with other
things thrown in. Or, a zipcode column should only generate a zipcode that's
legal for the state.

# BUGS/SUGGESTIONS

This module is hosted on Github at
[https://github.com/robkinyon/dbix-class-sims](https://github.com/robkinyon/dbix-class-sims). Pull requests are strongly
encouraged.

# DBIx::Class::Fixtures

[DBIx::Class::Fixtures](http://search.cpan.org/perldoc?DBIx::Class::Fixtures) is another way to load data into a database. Unlike
this module, [DBIx::Class::Fixtures](http://search.cpan.org/perldoc?DBIx::Class::Fixtures) approaches the problem by loading the same
data every time. This is complementary because some tables (such as lookup
tables of countries) want to be seeded with the same data every time. The ideal
solution would be to have a set of tables loaded with fixtures and another set
of tables loaded with sims.

# SEE ALSO

[DBIx::Class](http://search.cpan.org/perldoc?DBIx::Class), [DBIx::Class::Fixtures](http://search.cpan.org/perldoc?DBIx::Class::Fixtures)

# AUTHOR

Rob Kinyon <rob.kinyon@gmail.com>

# LICENSE

Copyright (c) 2013 Rob Kinyon. All Rights Reserved.
This is free software, you may use it and distribute it under the same terms
as Perl itself.
