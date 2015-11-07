#EXCEL FORMULAS

##DATA MERGING

1. Merge two columns with different cells of data into a new single column.
Add into new blank column. Formula states: IF B2 is blank, concatenate B2 and Z2 *(trick to keep Excel from adding a 0)* ELSE insert B2

```python
=IF(ISBLANK($B2),$B2&$Z2,$B2)
```

2. Merge two cells into one if neither are blank

```python
=IF(ISBLANK($L2),"",IF(ISBLANK($R2),"",$L2&" "&$R2))
```

#Nested IF Statements

```python
=IF($L3="female","F",IF($L3="Female","F",IF($L3="male","M",IF($L3="Male","M",$L3))))
```




