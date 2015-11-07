# RSForm (v1.50) Image Uploading Guide

##Obtain the value of the File Field
### From the $POST/$FILES Obect

```php
// get file size in kb
var_dump($_FILES['form']['size']['<FORM_FIELD_ID>']);

// get file error number
var_dump($_FILES['form']['error']['<FORM_FIELD_ID>']);
```

### From RSForm Placeholder Tags
```bash
{<FORM_FIELD_ID>:caption}
{<FORM_FIELD_ID>:description}
{<FORM_FIELD_ID>:name}
{<FORM_FIELD_ID>:value}
{<FORM_FIELD_ID>:path}
{<FORM_FIELD_ID>:localpath}
{<FORM_FIELD_ID>:filename}
```

Return the value of a placeholder tag in PHP
  
```php
list($replace, $with) = RSFormProHelper::getReplacements($SubmissionId);
$uploadName = str_replace($replace, $with, '{strPhoto:filename}');
var_dump($uploadName);
exit;
```
 
## Modify File Names on Upload
 
 Add the following to the "Script called on form process" section in RSForm's PHP Scripts section.
 
 ```php
 // Lowercase filename and replace spaces with dashes_
$filename = $_FILES['form']['name']['strPhoto'];
$lcfilename = strtolower(str_replace(' ', '-', $filename));
$_FILES['form']['name']['strPhoto'] = $lcfilename;
//var_dump($_FILES['form']['name']['strPhoto']);
//exit;
 ```

## Resizng Images
One possible approach is to add something like this in the 'after form has been processed' of the PHP Scripts section:

```php
// Look for files in the holding area then move and resize any we find.
$tempfiles = glob('/home/jdoyle/www/alaskamagazine/images/contests/photocontest2015/subs/*.{jpeg,jpg,gif,png}', GLOB_BRACE);

foreach($tempfiles as $file) {

	// Make a copy of the original file for safe keeping
	$pathparts = parse_url(strtolower($file));
	$fpath = pathinfo($pathparts['path'], PATHINFO_DIRNAME);
	$fname = pathinfo($pathparts['path'], PATHINFO_FILENAME);
	$fext = pathinfo($pathparts['path'], PATHINFO_EXTENSION);
	
	$ofile = $fpath . "/orig/" . $fname . "_orig" . "." . $fext;
	copy($file, $ofile);
	
	// Now let's resize the image for web
	$ratio = 0;
	list($width, $height) = getimagesize($file);

	if($width > 1920 || $height > 1200) {
		if($width > 1920) {
			$ratio = 1920/$width;
		} else if($height > 1200) {
			$ratio = 1200/$height;
		}
	}

	switch($fext) {
		case "jpg":
		$tcimg = imagecreatetruecolor($width*$ratio, $height*$ratio);
		$img = imagecreatefromjpeg($file);
		imagecopyresized($tcimg, $img, 0,0,0,0, $width*$ratio, $height*$ratio, $width, $height);
		imagejpeg($tcimg, $file, 80);
		break;
	}

}


```

