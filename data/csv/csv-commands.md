#Working CSVs from the Command Line

## Data Normalization

###Remove Header

```bash
sed file2.csv > file2_trimmed.csv
```

```bash
# Alternative approach to removing CSV headers
cat input.csv | sed "1 d" > noheader.csv
```

###Concatenate Multiple CSV Files

```bash
cat *.csv > output.csv
```
----


## Data Cleansing


###Remove Duplicate Lines with Sort and Uniq

```bash
sort {file_name} | uniq -u
```

###Combine Multiple Files and Dedupe

```bash
cat file1.csv file2.csv | sort combined.csv | uniq -u
cat file1.csv file2.csv | sort | uniq -u > nbsa_deduped.csv
```
----


## Command Reference

###Unique 

```bash
unique -uc -f 5-12
```

- -u : list only the unique lines
- -d : list only the duplicate lines
- -c : include count
- -f num : skip fields

## Reference
###Python Commands CSV Reference
[http://bconnelly.net/working-with-csvs-on-the-command-line/](http://bconnelly.net/working-with-csvs-on-the-command-line)
