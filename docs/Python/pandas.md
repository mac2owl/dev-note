# Pandas

## Flatten entity–attribute–value models/tables using Pandas dataframe

users (entity/object table)

| id  | name       | email                  |
| --- | ---------- | ---------------------- |
| 1   | John Smith | john.smith@example.com |
| 2   | Jane Smith | jane.smith@example.com |

page_config (attribute table)

| id  | name             |
| --- | ---------------- |
| 1   | font-size        |
| 2   | color            |
| 3   | background-color |

users_page_config (EAV table)

| id  | user_id | page_config_id | value   |
| --- | ------- | -------------- | ------- |
| 1   | 1       | 1              | 12px    |
| 2   | 1       | 2              | #3e3e3e |
| 3   | 1       | 3              | blue    |
| 4   | 2       | 1              | 16px    |
| 5   | 2       | 2              | #000000 |
| 6   | 2       | 3              | green   |

```py
import numpy as np
import pandas as pd


class EavFlattener:
	def flatten_eav_table(self, data):
		result_df = pd.DataFrame(data)
		result_df = result_df.pivot(index="user_id", columns='attribute')['attribute_value']
		result_df = result_df.reset_index()
		result_df = result_df.drop(np.nan, axis=1)
		return result_df


data = [
	{'user_id': 1, 'attribute': 'font-size', 'attribute_value': '12px'},
	{'user_id': 1, 'attribute': 'color', 'attribute_value': '#3e3e3e'},
	{'user_id': 1, 'attribute': 'background-color', 'attribute_value': 'blue'},
	{'user_id': 2, 'attribute': 'font-size', 'attribute_value': '16px'},
	{'user_id': 2, 'attribute': 'color', 'attribute_value': '#000000'},
	{'user_id': 2, 'attribute': 'background-color', 'attribute_value': 'green'}
]

EavFlattener().flatten_eav_table(data)

# Flattened EAV data
# [{'user_id': 1, 'background-color': 'blue', 'color': '#3e3e3e', 'font-size': '12px'},
#  {'user_id': 2, 'background-color': 'green', 'color': '#000000', 'font-size': '16px'}]
```

## Parquet with selected columns

```py

import pandas as pd
import pyarrow.parquet as pq

parquet_file = pq.ParquetFile(file_name)
selected_columns = [...]
selected_columns = [col for col in selected_columns if col in parquet_file.schema.names]
pq_filters =[("email", "in", user_emails), ("Year", "in", years)]
df = pd.read_parquet(
		file_name,
		engine="pyarrow",
		columns=selected_columns,
		filters=pq_filters,
)

...


```
