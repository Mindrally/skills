---
name: analytics-data-analysis
description: "Best practices for analytics, data analysis, and visualization using Python, pandas, matplotlib, seaborn, and Jupyter notebooks. Use when performing exploratory data analysis, building data pipelines, creating statistical visualizations, writing Jupyter notebooks, cleaning and transforming datasets, or implementing analytics dashboards."
metadata:
  maintainer: Mindrally
  source: https://github.com/Mindrally/skills
---

# Analytics and Data Analysis

Guidelines for data analysis, visualization, and Jupyter-based workflows using pandas, matplotlib, seaborn, and numpy. Prioritize readability, reproducibility, and vectorized operations.

## Workflow: Exploratory Data Analysis Pipeline

1. **Load and inspect** — Read data with `pd.read_csv()` or appropriate loader, check `.shape`, `.dtypes`, `.describe()`, and `.isnull().sum()`
2. **Clean and transform** — Handle missing values, fix dtypes, rename columns, filter outliers using vectorized pandas operations
3. **Explore relationships** — Use `.groupby()`, `.corr()`, and cross-tabulations to identify patterns
4. **Visualize findings** — Create targeted plots with matplotlib/seaborn; label axes, add titles, use colorblind-friendly palettes
5. **Validate results** — Run statistical tests, report confidence intervals, verify assumptions
6. **Document and share** — Structure notebook with markdown sections, clear outputs before sharing, pin dependencies

## Key Principles

- Write concise, technical code with accurate Python examples
- Emphasize readability and reproducibility in data analysis workflows
- Use functional programming patterns; minimize class usage
- Leverage vectorized operations over explicit loops for performance
- Use descriptive variable naming conventions (e.g., `is_valid`, `has_data`, `total_count`)
- Adhere to PEP 8 style guidelines

## Quick Start Example

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

# Load and inspect
df = pd.read_csv("data.csv", parse_dates=["timestamp"])
print(f"Shape: {df.shape}, Missing: {df.isnull().sum().sum()}")

# Clean: drop rows missing target, fill numeric gaps with median
df = (
    df.dropna(subset=["revenue"])
    .assign(category=lambda x: x["category"].astype("category"))
    .fillna(df.select_dtypes("number").median())
)

# Analyze: revenue by category
summary = df.groupby("category")["revenue"].agg(["mean", "median", "std"])

# Visualize
fig, ax = plt.subplots(figsize=(10, 6))
sns.boxplot(data=df, x="category", y="revenue", palette="colorblind", ax=ax)
ax.set_title("Revenue Distribution by Category")
ax.set_ylabel("Revenue ($)")
plt.tight_layout()
plt.savefig("revenue_by_category.png", dpi=150)
plt.show()
```

## Data Analysis with Pandas

### Data Manipulation Best Practices
- Use pandas for all data manipulation and analysis tasks
- Apply method chaining for clean, readable transformations
- Utilize `loc` and `iloc` for explicit data selection
- Employ `groupby` for efficient data aggregation
- Use `merge` and `join` appropriately for combining datasets

### Performance Optimization
- Use vectorized operations instead of loops
- Utilize efficient data structures like categorical data types for low-cardinality string columns
- Consider dask for larger-than-memory datasets
- Profile code to identify and optimize bottlenecks
- Use appropriate dtypes to minimize memory usage

### Data Validation
- Validate data types and ranges to ensure data integrity
- Use try-except blocks for error-prone operations when reading external data
- Check for missing values and handle appropriately
- Verify data shape and structure after transformations

## Visualization Standards

### Matplotlib Guidelines
- Use matplotlib for fine-grained customization control
- Create clear, informative plots with proper labeling
- Always include axis labels and titles
- Use consistent color schemes across related visualizations
- Save figures with appropriate resolution for the intended use

### Seaborn for Statistical Visualizations
- Apply seaborn for statistical visualizations and attractive defaults
- Leverage built-in themes for consistent styling
- Use appropriate plot types for the data (scatter, line, bar, heatmap, etc.)
- Consider color-blindness accessibility in color palette choices

### Accessibility in Visualizations
- Use colorblind-friendly palettes
- Include alternative text descriptions
- Ensure sufficient contrast in visual elements
- Provide data tables as alternatives to complex charts

## Jupyter Notebook Best Practices

### Notebook Structure
- Structure notebooks with clear markdown sections
- Begin with an overview/introduction cell
- Document analysis steps thoroughly
- Keep code cells focused and modular
- End with conclusions and key findings

### Execution and Reproducibility
- Maintain meaningful cell execution order
- Clear outputs before sharing notebooks
- Use environment files (requirements.txt) for dependencies
- Document data sources and access methods
- Include date/version information

### Code Organization
- Import all libraries at the notebook beginning
- Define helper functions in dedicated cells
- Use magic commands appropriately (%matplotlib inline, etc.)
- Keep individual cells concise and single-purpose

## Technical Requirements

### Core Dependencies
- pandas: Data manipulation and analysis
- numpy: Numerical computing
- matplotlib: Base plotting library
- seaborn: Statistical data visualization
- jupyter: Interactive computing environment

### Extended Libraries
- scikit-learn: Machine learning tasks
- scipy: Scientific computing
- plotly: Interactive visualizations
- statsmodels: Statistical modeling

## Analytics Implementation

### Tracking and Measurement
- Define clear metrics and KPIs before analysis
- Document data collection methodology
- Implement proper data pipelines for reproducibility
- Create automated reporting where appropriate
- Version control notebooks and analysis scripts

### Statistical Analysis
- Use appropriate statistical tests for the data type
- Report confidence intervals alongside point estimates
- Be cautious about p-value interpretation
- Consider effect sizes, not just statistical significance
- Document assumptions and limitations

## Error Handling and Logging

- Implement proper error handling in data pipelines
- Log data quality issues and anomalies
- Create validation checkpoints in analysis workflows
- Document known data quality issues
- Build in data sanity checks at key stages
