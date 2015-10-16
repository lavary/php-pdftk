# Filling Out PDF Forms With PDFtk


## Introduction

PDF files are nowadays one of the most common ways of sharing documents on Internet. Whether you need to pass your clients' documents to third-party service providers like banks or insurance companies, or just to send your CV to employers, using a PDF document is always the first option.

PDF files can transfer plain/formatted texts, images, hyperlinks and even fillable forms. In this tutorial, we're going to see how we can fill out PDF forms programmatically, with PHP and a great PDF manipulation tool called **PDFtk Server**.

To keep things simple enough, we'll refer to `PDFtk Server` as `PDFtk` through the rest of the article. 

## Installation

### Install PDFtk on Ubuntu:

You can install `PDFtk` on Ubuntu via `apt-get` package manager. If you don't have access to a linux environment, you may use the [Homestead Improved Vagrant VM](https://github.com/Swader/homestead_improved/) to set up you own linux based development environment. This [quick tip](http://www.sitepoint.com/quick-tip-get-homestead-vagrant-vm-running/) will help you get up and running with a brand new Homestead Improved Vagrant VM.

Once the VM is booted and you've successfully managed to ssh into the system, you may start installing `PDFtk` using `apt-get`:

```
sudo apt-get install pdftk
```  

To check if it works on your machine, run the following command:

```
pdftk --version
```

You should get something like the following output:

```
Copyright (c) 2003-13 Steward and Lee, LLC - Please Visit:www.pdftk.com. This is free software; see the source code for copying conditions. There is NO warranty, not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

For installing PDFtk on other machines, you may visit the [`PDFtk Server`](https://www.pdflabs.com/tools/pdftk-server/) download page. It has all the required information to get you up and running in no time.

## How It Works

PDFtk provides a wide variety of features for manipulating PDF documents from merging and splitting pages to filling out PDF forms or even applying watermarks. In this article, we're going to use PDFtk to fill out a standard PDF form dynamically using PHP.

 PDFtk uses **FDF** files for manipulating PDF forms. FDF or **Form Data File** is a plain-text file, which can store form submitted data in a much simpler structure than PDF files.

Simply put, we need to generate an FDF file from **user submitted data**, and merge it with the original PDF file using PDFtk's commands.

### What Is Inside an FDF File

The structure of an FDF file is composed of three parts: header, content and footer: 

**Header**

```
%FDF-1.2
1 0 obj<</FDF<< /Fields[
```

We don't need to worry about this part since it's what we're going to use for all FDF files.

**Content**

A simple FDF content is as follows:

```
<< /T (first_name) /V (John)
<< /T (last_name) /V (Smith)
<< /T (occupation) /V (Teacher)>>
<< /T (age) /V (45)>>
<< /T (gender) /V (male)>>
```

The content section may seem confusing at start, but don't worry, we'll get to that shortly.

**Footer**

```
] >> >>
endobj
trailer
<</Root 1 0 R>>
%%EOF
```

This section is also the same for all FDF files.

The content section is consisted of form data entries, each following the same pattern. Each line represents one field in the form. They begin with the element's name prefixed with `\T`, which indicates the title. The second part is the element's value prefixed with `\V` indicating the value:

```
<< /T(FIELD_NAME)/V(FIELD_VALUE) >>
```

In order to create an FDF file, we will need to know the field names in the PDF form. If you have access to a Mac or Windows machine, you may open the form in Adobe Acrobat Pro and see the field properties.

Alternatively we can use PDFtk's `dump_data_fields` command to extract fields information from the file:

```
pdftk path/to/the/form.pdf dump_data_fields > field_names.txt
```

As a result, the output will be saved in `field_names.txt`. Below is an example of the extracted data:

```
--
FieldType: Text
FieldName: first_name
FieldFlags: 0
FieldJustification: Left
---
FieldType: Text
FieldName: last_name
FieldFlags: 0
FieldJustification: Left
---
FieldType: Text
FieldName: occupation
FieldFlags: 0
FieldJustification: Center
---
FieldType: Button
FieldName: gender
FieldFlags: 0
FieldJustification: Center
```

As you can see, there are several properties for each field in the form. This values can be modified in Adobe Acrobat Pro. For example you can change text alignments, font sizes or even the text color.

### PDFtk and PHP

We can use PHP's `exec()` function to bring PDFtk to the PHP environment. Let's write a simple script to fill out a form.

Suppose we have a simple PDF form with 4 text boxes and a group of two radio buttons:

![Blank PDF form]
(raw_form.jpg)

Launch your favorite text editor and paste in the following code:

```php

<?php

$fname      = 'John';
$lname      = 'Smith';
$occupation = 'Teacher';
$age        = '45';
$gender     = 'male';

$fdf_header = <<<FDF
%FDF-1.2
%,,oe"
1 0 obj
<<
/FDF << /Fields [
FDF;

$fdf_footer = <<<FDF
"] >> >>
endobj
trailer
<</Root 1 0 R>>
%%EOF;
FDF;

// FDF field values

$fdf_content  = "<</T(first_name)/V({$fname})>>";
$fdf_content .= "<</T(last_name)/V({$lname})>>";
$fdf_content .= "<</T(occupation)/V({$occupation})>>";
$fdf_content .= "<</T(age)/V({$age})>>";
$fdf_content .= "<</T(gender)/V({$gender})>>";

$content = $fdf_header . $fdf_content , $fdf_footer;

// Creating a temporary file for our FDF file.
$FDFfile = tempnam(sys_get_temp_dir(), gethostname());

file_put_contents($FDFfile, $content);

exec("pdftk form.pdf fill_form $FDFfile output.pdf"); 

unlink($FDFfile);

```

Okay, let's break the script down. At First, we defined the values that we're going to write to the form. These values can be fetched from a database table, a JSON API response or even hardcoded in the file, as we just did for this example.

Next we created an FDF file based on the pattern described earlier.  I used PHP's [tempnam](http://php.net/manual/en/function.tempnam.php) function to create a temporary file for storing the FDF content. The reasons is that PDFtk only relies on physical files to perform the operations, specially when filling out forms.

Finally, we called the PDFtk's `fill_form` command using PHP's `exec` function. `fill_form` **merges** the FDF file with the raw PDF form. According to the script, our PDF file should be in the same directory of our script. However this path can be modified based on our needs.

Save the file in your web root directory as `pdftk.php` or whatever name that feels right to you. The result will be a new PDF file with all the fields filled out with our data.

![Filled PDF form]
(filled_form.jpg)

As simple as that!

### Flattening the Output File

The output file can be flattened to prevent future modifications. This is done by passing `flatten` as a parameter to `fill_form` command.

```php
<?php
exec("pdftk path/to/form.pdf fill_form $FDFfile output path/to/output.pdf flatten"); 
```

### Downloading the Output File

Instead of storing the file on the disk, we can force download the output file by sending the the file's content along with the required headers to the output buffer:

```php
<?php

// ...

exec("pdftk path/to/form.pdf fill_form $FDFfile output output.pdf flatten"); 


// Force Download the output file
header('Content-Description: File Transfer');
header('Content-Type: application/octet-stream');
header('Content-Disposition: attachment; filename=' . 'path/to/output.pdf' );
header('Expires: 0');
header('Cache-Control: must-revalidate');
header('Pragma: public');
header('Content-Length: ' . filesize('output.pdf'));

readfile('output.pdf');            

exit;

```
Now, if we run the script again, the output file will be automatically downloaded.

Now that we have a basic understanding of how PDFtk works, we can start building a PHP class around it, to make our service more reusable.

## Creating a Wrapper Class Around PDFtk

Before getting our hands wet and start building the class, let's briefly review the steps we have to take, when filling out a PDF form using PDFtk:

1. Knowing the name of the PDF form's elements in advance
2. Preparing the values to be written to the form, either from a database, HTML post or a RESTful API
3. Generating the FDF file
4.  Merging the FDF file with the blank PDF form using PDFtk's `fill_form` command
5.  Save or download the output file

The usage of our final product should be as simple as the following code:

```php
<?php

// Data to be written to the PDF form
$data = [
    'first_name' => 'John',
    'last_name'  => 'Smith',
    'occupation' => 'Teacher',
    'age'        => '45',
    'gender'     => 'male'
];

$pdf = new pdfForm('form.pdf', $data);

$pdf->flatten()
    ->save('outputs/form-filled.pdf')
    ->download();

```

### Creating the Class

Open your favorite text editor, create a new file and name it **pdfForm.php**. Let's name the class `pdfForm` as well. 

#### Starting With Class Properties

First of all, we need to declare some private properties for the class:

```php
<?php

class pdfForm {
/*
 * Path to the raw PDF form
 * @var string
 */
private $pdfurl;

/*
 * Form data
 * @var array
 */
private $data;

/*
 * Path to the output file
 * @var string
 */
private $output;

/*
 * A flag for flattening the output file
 * @var boolean
 */ 
private $flatten;

// ...

}
```
 

#### The Constructor

Ok now let's create the constructor method:

```php
<?php

// ...

function __construct($pdfurl, $data) {
        $this->pdfurl = $pdfurl;
        $this->data   = $data;
}
```

The constructor doesn't do anything complicated except for assigning PDF path and the form data to their respective properties inside the class.

#### Handling Temporary Files

Since PDFtk heavily makes use of physical files to perform its tasks, we usually need to generate temporary files during the process. In order to keep the code clean and reusable, let's create a method which creates temporary files:

```php
<?php

// ...

private function _tmpfile() {
    return tempnam(sys_get_temp_dir(), gethostname());
}
```

As a convention, let's put an underscore prefix to each private method's name, just to indicate it is a private method. This method actually creates a file using PHP's [tempnum](http://php.net/manual/en/function.tempnam.php) function. As you can see, we passed two parameters to the function, the first parameter is the path to the `tmp` directory which is get via [sys_get_temp_dir](http://php.net/manual/en/function.sys-get-temp-dir.php) function. The second parameter is a prefix for the filename, just to make sure the filename would be as unique as possible. In this case, the file name will be prefixed with our host name. The method returns the file path to the caller.


#### Extracting Form Information

As noted earlier, to create an FDF file, we need to know the name of the fields in advance. This is possible either by opening the form in **Adobe Acrobat Pro** or alternatively by using PDFtk's  `dump_data_fields` command. 

To make things easier for the developer, let's create a method which prints out field information to the screen. Although this method won't be used in the PDF generation process, but can be useful when we're not aware of the field names. Another use case would be parsing fields meta data to make the writing process more dynamic.

```php
<?php
public function fields($pretty = false) {
        $tmp = $this->_tmpfile();

        exec("pdftk {$this->pdfurl} dump_data_fields > {$tmp}"); 
        $con  = file_get_contents($tmp);
        
        unlink($tmp);
        return ($pretty == true) ? nl2br($con) : $con;
    }
```

The above method, simply runs the PDFtk's `dump_data_fields` command, writes the output to a file and returns its contents. 

We also set an optional argument for beautifying the output. As a result we'll be able to get a human friendly output by passing `true` to the method. If you need to parse the output or run a regular expression against it,  we can call it without arguments.


#### Creating the FDF File 

In the next step, we will create a method for generating the FDF file:

```php
<?php
public function make_fdf($data) {
        
        //FDF header
        $fdf = '%FDF-1.2
1 0 obj<</FDF<< /Fields[';
        
        // FDF Content
        foreach($data as $key => $value) {
            $fdf .= '<</T(' . $key . ')/V(' . $value . ')>>';
        }
        
        // FDF footer
        $fdf .= "] >> >>
endobj
trailer
<</Root 1 0 R>>
%%EOF";

        // Creating a temporary file
        $fdf_file = $this->_tmpfile();
        file_put_contents($fdf_file, $fdf);     
        
        return $fdf_file;
    }
```

As you probably remember from the previous part, every FDF file is composed of header, content and footer sections.

Header and footer parts as it is obvious in the code, are static contents and need no explanation.

`make_fdf` method iterates over the `$data` array items to generate the entries based on the same pattern. Finally it puts the content to a temporary file using `file_put_contents` and returns the file path to the caller.

#### Filling Out the Form

Now that we're able to create an FDF file, we can easily  fill the form using the `fill_form` command`:

```php
<?php
private function _generate() {
        
        $fdf = $this->make_fdf($this->data);         
        $this->output = $this->_tmpfile();
        exec("pdftk {$this->pdfurl} fill_form {$fdf} output {$this->output} {$this->flatten}");

        unlink($fdf);
    }

```

`generate()` calls `make_fdf()` method to generate the FDF file, then it runs the `fill_form` command to merge it with the raw PDF form.

The output is saved to a temporary file that we created with the `_tempfile()` method.

#### Flattening the File

We should be able to flatten the output file to prevent future modifications. PDFtk's `fill_form` accepts a `flatten` parameter for flattening the output file.

To do this, let's create a method to set the `$flatten` attribute to `flatten`.  This value is used by the `_generate()` method:

```php
<?php
 
// ...
public function flatten() {
    $this->flatten = 'flatten';
    return $this;
    }
```

#### Saving the File

When the file is generated, we might want to save or download it or do both at the same time.

First let's create the saver method:

```php
<?php
public function save($path = null) {
        if(is_null($path)) {
            return $this;
        }

        if(!$this->output) {
            $this->_generate();
        }

        $dest = pathinfo($path, PATHINFO_DIRNAME);
        if(!file_exists($dest)) {
            mkdir($dest, 0775, true);
        }

        copy($this->output, $path);
        unlink($this->output);
        
        $this->output = $path;

        return $this;
    }
```

At first, the method checks if there's any path given for the destination; If the destination path is null it just returns without saving the file, otherwise it will proceed to the next part.

Next, it checks if the file has been already generated; If it's not,  It will call `_generate()` to generate it.

After making sure the output file is generated, it checks if the destination path exists on the disk. If the path doesn't exist, it will create the directories and sets the proper permissions.

In the end, the file is copied (from the `tmp` directory) to a permanent location and `$this->output` is updated to the permanent path.

#### Force Download the File

To force download the file, we need to send the file's content along with the required headers to the output buffer.

```php
<?php
public function download() {
        
        if(!$this->output) {
            $this->_generate();
        }
        
        $filepath = $this->output;
        if( file_exists($filepath) ) {        
            
            header('Content-Description: File Transfer');
            header('Content-Type: application/pdf');
            header('Content-Disposition: attachment; filename=' . uniqid(gethostname()) . '.pdf' );
            header('Expires: 0');
            header('Cache-Control: must-revalidate');
            header('Pragma: public');
            header('Content-Length: ' . filesize($filepath));
            
            readfile($filepath); 

            exit;
        }
    }
```

In this method, first we need to check if the file has been generated because we might need to download the file without saving it. After making sure that everything is set, we can send the file's content to output buffer using PHP's `readfile()` function.

And that is all there is to it! Our pdfForm class is ready to use now. You can get the full code on [GitHub](https://github.com/lavary/pdf-form)

### Putting the Class Into Action

To use the class, first we need to make sure it is loaded before using it either by using an autoloader or simply using the `require` directive.

####Filling Out the Form

```php
<?php
require 'pdfForm.php';

$data = [
    'first_name' => 'John',
    'last_name'  => 'Smith',
    'occupation' => 'Teacher',
    'age'        => '45',
    'gender'     => 'male'
];

$pdf = new pdfForm('form.pdf', $data);

$pdf->flatten()
    ->save('output.pdf')
    ->download();
```

Data can be fetched from different sources like a database table, a JSON object or just an array as we did in above snippet.

####Creating an FDF File

If we just need to create an FDF file without filling out a form, we can only use `make_fdf()` method.

```php
<?php
require 'pdfForm.php';

$data = [
    'first_name' => 'John',
    'last_name'  => 'Smith',
    'occupation' => 'Teacher',
    'age'        => '45',
    'gender'     => 'male'
];

$pdf = new pdfForm('form.pdf', $data);

$fdf = $pdf->make_fdf();
```
The return value of `make_fdf()` is the path to the generated FDF file in the `tmp` directory. You can either get the content of the file or save it to a permanent location.

####Extracting PDF Field Information

If we just need to see what fields and field types exist in the form,  we can call the `fields()` method to get the information:

```php
<?php

require 'pdfForm.php';

$fields = new pdfForm('form.pdf')->fields();

echo $fields;

```

If there's no need to parse the output, we can pass `true` to the `fields()` method to get a human readable output:

```php
<?php

require 'pdfForm.php';

$pdf = new pdfForm('pdf-test.pdf')->fields(true);

echo $pdf;

```

## Wrapping up

We installed `PDFtk` and learned some of its useful commands like `dump_data_fields` and `fill_form`. Then we created very basic class around PDFtk to demonstrate how we can bring PDFtk's power to our PHP applications. You can extend this class or create one from scratch using the same concept.

Please note that this implementation is very basic and we tried to keep things as bare bones as possible; To have a more elaborate code, you can go further and put the FDF creation feature in a separate class; This approach gives you more space when working with FDF files . For example you can easily add filters to each form data entry like upper case, lower case or even format a date, just to name a few. In addition to that, you can implement a `download()` and a `save()` methods for the FDF class as well.

Hope this tutorial solves a problem.

Thanks for reading.
