import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LinearRegression, Ridge
from xgboost import XGBRegressor
from sklearn.metrics import mean_squared_error
from statsmodels.stats.outliers_influence import variance_inflation_factor
import statsmodels.api as sm

# Load the data
data_file_path = '/content/imports-85.data'

# Column names
column_names = [
    "symboling", "normalized_losses", "make", "fuel_type", "aspiration", "num_doors", "body_style",
    "drive_wheels", "engine_location", "wheel_base", "length", "width", "height", "curb_weight",
    "engine_type", "num_cylinders", "engine_size", "fuel_system", "bore", "stroke", "compression_ratio",
    "horsepower", "peak_rpm", "city_mpg", "highway_mpg", "price"
]

# Load the data
df = pd.read_csv(data_file_path, names=column_names)

# Data Cleaning
df.replace("?", pd.NA, inplace=True)
df['normalized_losses'] = pd.to_numeric(df['normalized_losses'], errors='coerce')
df['bore'] = pd.to_numeric(df['bore'], errors='coerce')
df['stroke'] = pd.to_numeric(df['stroke'], errors='coerce')
df['horsepower'] = pd.to_numeric(df['horsepower'], errors='coerce')
df['peak_rpm'] = pd.to_numeric(df['peak_rpm'], errors='coerce')
df['price'] = pd.to_numeric(df['price'], errors='coerce')

# Impute missing values with median for numerical columns
df['normalized_losses'].fillna(df['normalized_losses'].median(), inplace=True)
df['bore'].fillna(df['bore'].median(), inplace=True)
df['stroke'].fillna(df['stroke'].median(), inplace=True)
df['horsepower'].fillna(df['horsepower'].median(), inplace=True)
df['peak_rpm'].fillna(df['peak_rpm'].median(), inplace=True)
df['price'].fillna(df['price'].median(), inplace=True)
df['num_doors'].fillna(df['num_doors'].mode()[0], inplace=True)

# Step 1: Handling Outliers in 'price'
# Removing outliers in the 'price' column by capping at the 95th percentile
cap_value = df['price'].quantile(0.95)
df['price'] = np.where(df['price'] > cap_value, cap_value, df['price'])

# Step 2: Reducing Multicollinearity by Dropping 'highway_mpg'
df_reduced = df.drop(columns=['highway_mpg'])

# Recalculating the VIF after dropping 'highway_mpg'
numerical_cols_reduced = numerical_cols.drop('highway_mpg')
X_reduced = df_reduced[numerical_cols_reduced].dropna()  # Drop NA values for VIF calculation
X_reduced = sm.add_constant(X_reduced)  # Add a constant for the intercept

vif_data_reduced = pd.DataFrame()
vif_data_reduced["Feature"] = X_reduced.columns
vif_data_reduced["VIF"] = [variance_inflation_factor(X_reduced.values, i) for i in range(X_reduced.shape[1])]

print(vif_data_reduced.sort_values(by="VIF", ascending=False))

# Exploratory Data Analysis (EDA)
# 1. Distribution of Car Prices
sns.histplot(df['price'], kde=True, color='blue')
plt.title('Distribution of Car Prices')
plt.xlabel('Price')
plt.ylabel('Frequency')
plt.show()

# 2. Scatter plot of Engine Size vs. Price
sns.scatterplot(x='engine_size', y='price', data=df, hue='fuel_type')
plt.title('Engine Size vs Price')
plt.xlabel('Engine Size')
plt.ylabel('Price')
plt.show()

# 3. Correlation Matrix
# Encode categorical columns using one-hot encoding
encoded_df = pd.get_dummies(df, drop_first=True)

# Compute the correlation matrix on the encoded dataset
corr_matrix = encoded_df.corr()

# Plot the correlation matrix
plt.figure(figsize=(14, 10))
sns.heatmap(corr_matrix, annot=True, fmt='.2f', cmap='coolwarm', square=True)
plt.title('Correlation Matrix of Encoded Features')
plt.show()

# Feature Preparation using the reduced dataset
features_reduced = df_reduced.drop(columns=['price'])
target_reduced = df_reduced['price']

# Identify categorical and numerical columns
categorical_cols_reduced = features_reduced.select_dtypes(include=['object']).columns
numerical_cols_reduced = features_reduced.select_dtypes(include=['number']).columns

# Preprocessing for categorical and numerical data
categorical_transformer_reduced = Pipeline(steps=[
    ('onehot', OneHotEncoder(handle_unknown='ignore'))
])
numerical_transformer_reduced = Pipeline(steps=[
    ('scaler', StandardScaler())
])
preprocessor_reduced = ColumnTransformer(
    transformers=[
        ('num', numerical_transformer_reduced, numerical_cols_reduced),
        ('cat', categorical_transformer_reduced, categorical_cols_reduced)
    ])

# Define the Ridge Regression pipeline
ridge_model = Pipeline(steps=[('preprocessor', preprocessor_reduced),
                              ('regressor', Ridge())])

# Split the data into training and testing sets
X_train_reduced, X_test_reduced, y_train_reduced, y_test_reduced = train_test_split(features_reduced, target_reduced, test_size=0.2, random_state=42)

# Train the Ridge Regression model
ridge_model.fit(X_train_reduced, y_train_reduced)

# Make predictions
ridge_preds = ridge_model.predict(X_test_reduced)

# Evaluate the Ridge Regression model
ridge_rmse = np.sqrt(mean_squared_error(y_test_reduced, ridge_preds))
print(f"Initial Ridge Regression RMSE: {ridge_rmse}")

# Hyperparameter tuning for Ridge Regression
# Define the parameter grid for Ridge Regression
ridge_param_grid = {
    'regressor__alpha': [0.01, 0.1, 1, 10, 100, 1000]
}

# Initialize GridSearchCV with Ridge Regression
ridge_grid_search = GridSearchCV(ridge_model, ridge_param_grid, cv=5, scoring='neg_mean_squared_error', n_jobs=-1)

# Fit the grid search
ridge_grid_search.fit(X_train_reduced, y_train_reduced)

# Best parameters and best score
best_ridge_params = ridge_grid_search.best_params_
best_ridge_rmse = np.sqrt(-ridge_grid_search.best_score_)

print(f"Best Ridge Parameters: {best_ridge_params}")
print(f"Best Ridge RMSE: {best_ridge_rmse}")
