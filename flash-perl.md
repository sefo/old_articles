# Interaction Flash/Perl

*Published 2005-01-17*

Wayback machine: https://web.archive.org/web/20140214173017/http://osix.net/modules/article/?id=651

> How can we use flash as an interface to a perl login script?
> And some other ActionScript/Perl tips.
> For this article I use SwishMax but any tool will do.

## 1/ Introduction:

This interaction opens a lot of possibilities! You can even have perl do what ActionScript can't do.
In ActionScript, LoadVariables(location) will get any number of variables returned by "location" (a perl script, php script or text file). These variables will be directly returned to your flash movie.

_Potential hosting problems_: The variables you load have to be **plain text**. Some free hosts will add banners to anything you put on the server. In this case, what's explained here will not work...

## 2/ Example 1: Hello World!

I write this example to explain how ActionScript works with external datas.
This script will take some value stored in a text file and display it.

### a/ Flash

Create a dynamic textbox (named "MyTxt" for example) and put hello as variable name.

_Note_: Note that in SwishMax you will have to check "target" in the textbox properties to be able to enter a variable name.

Then create a button and add the following function to its onRelease() event:

`loadVariables("http://yourserver/textfile.txt")`

This function will get any variable that's inside the text file and directly put it in the MyTxt textbox. You don't have to specify any variable in your ActionScript!

### b/ Text file

A valid variable definition looks like:
`&variable=value`

Then in our text file we will add:
`&hello=Hello+World!`

_hello_ is the variable name of our textbox and it's also the variable name of the message we want to display. flash will do the rest and transfer the value from the text file into the textbox.

_note:_ Note that `<space>` is replaced by `+`


## 3/ Example 2: Hello World 2!

This script will ask perl for a string and put it in our textbox.

### a/ Flash

You just have to keep the flash movie you created in example 1 and change the function to:

`loadVariables("http://yourserver/cgi-bin/hello.pl")`

### b/ Perl

Now make sure that the perl script returns a string formatted like the one we added in the text file of example 1.

```
#!/usr/bin/perl
print "Content-type:text/plain\\n\\n";
print "&hello=Hello+World+2!";
exit 0;
```

Here we need plain text because Flash won't recognize html.
The returned value of the perl script will be `&hello=Hello+World+2!`
and Flash will do the rest exactly like in example 1.

## 4/ Example 3: Login Script

Now to make the explanation easier (and for test purpose), we will create a plain text database. (You can use any other database if you like).

The perl script will look inside this database and look for a name:password pair, compare it with the informations entered in the flash movie and return a value ok/wrong.

### a/ Database

Create a text file "database.txt" which looks like this:

```
sefo:abcd
robert:efgh
admin:admin
```

### b/ Perl

Now the script will get name and password parameters with the GET method and do the comparison:

```
#!/usr/bin/perl

use CGI qw/:all/;

my $name=param('Name');
my $pass=param('Pass');

my $db="http://yourserver/database.txt";
my $good=0;

open(FILE,"$db") || die "Error: $!\\n";
flock(FILE, 2) || die("Can't flock file - $!\\n");
chomp(@lines = <FILE>);

foreach my $line (@lines) {
  my ($rec1,$rec2)=split(/:/,$line);
    if ($name eq $rec1) {
      if ($pass eq $rec2) {
        $good=1;
        print "Content-type:text/plain\\n\\n";
        print "&result=Login+OK!";
        close (FILE);
        exit 0;
      }
    }
}
close (FILE);
if ($good!=1) {
        print "Content-type:text/plain\\n\\n";
        print "&result=Wrong+Login!";
}
exit 0;
```

You can call the script with:
`http://yourserver/cgi-bin/hello.pl?Name=admin&Pass=admin`

The script opens the database, reads each line and separates each name/password pair.
`my ($rec1,$rec2)=split(/:/,$line);`
Here rec1 will contain the name and rec2 the password.

The returned value (variable to use in flash) will be either "&result=Wrong+Login!" or "&result=Login+OK!"

### c/ Flash

We will need a textbox with variable name result to get the returned value.

Create one Dynamic textbox with name="MyResult", variable name="result" and text="Login Result"
Create 2 input texts with respective variable names: "txtName" and "txtPass"
And finally create one button with a script onRelease():

```
on(release) {
  pass=txtPass; //inputbox txtPass
  name=txtName; //inputbox txtName
  loadVariablesNum("http://yourserver/cgi-bin/hello.pl?Name="+name+"&Pass="+pass,"GET")
}
```

When the button is released, the variables pass and name will get the value of your two input boxes.
Then we use the function _loadVariablesNum()_ which accepts the path to the perl script + the method of transmission.
As we saw above, we have to call the perl script with 2 parameters which will be the values of our input boxes:

`http://yourserver/cgi-bin/hello.pl?Name=name&Pass=pass`

Perl will then return only one variable which is _&result=_
The content of this variable (Login OK or wrong Login) will be displayed in the textbox named **result**.
