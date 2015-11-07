# RSForm Placeholder Guide v1.50

##Possible placeholder values
### Text Boxes, Submit Buttons

```bash
{<FORM_FIELD_ID>:caption}
{<FORM_FIELD_ID>:description}
{<FORM_FIELD_ID>:name}
{<FORM_FIELD_ID>:value}
```

### Files

```bash
{<FORM_FIELD_ID>:caption}
{<FORM_FIELD_ID>:description}
{<FORM_FIELD_ID>:name}
{<FORM_FIELD_ID>:value}
{<FORM_FIELD_ID>:path}
{<FORM_FIELD_ID>:localpath}
{<FORM_FIELD_ID>:filename}
  ```
  
### Checkboxes, Radio Buttons

```bash
{<FORM_FIELD_ID>:caption}
{<FORM_FIELD_ID>:description}
{<FORM_FIELD_ID>:name}
{<FORM_FIELD_ID>:value}
{<FORM_FIELD_ID>:text}
```

### Global

```bash
{_STATUS:value}
{_ANZ_STATUS:value}
{global:username}
{global:userid}
{global:useremail}
{global:fullname}
{global:userip}
{global:date_added}
{global:sitename}
{global:siteurl}
{global:confirmation}
{global:submissionid}
{global:submission_id}
```

## Getting a Placeholder Value in Code
Should you need to, you can get the value of a place holder in PHP. For example, you can place something similar to this in the PHP Scripts - After Form Process section to return the filename of a file added to a form

```php
list($replace, $with) = RSFormProHelper::getReplacements($SubmissionId);
$uploadName = str_replace($replace, $with, '{strPhoto:filename}');
var_dump($uploadName);
exit;
```
