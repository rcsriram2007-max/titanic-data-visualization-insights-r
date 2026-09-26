

pkgs <- c("titanic", "tidyverse", "lattice", "scales", "gridExtra", "viridis")
for (p in pkgs) {
  if (!requireNamespace(p, quietly = TRUE)) install.packages(p)
}
library(titanic)
library(tidyverse)
library(lattice)
library(scales)
library(gridExtra)
library(viridis)

# Create output directory for figures if it doesn't exist
if (!dir.exists("outputs/figures")) dir.create("outputs/figures", recursive = TRUE)

# ------------------------------------------------------------------------------
# 1. Data Ingestion & Clean Baseline Construction
# ------------------------------------------------------------------------------
data("titanic_train", package = "titanic")
df <- as_tibble(titanic_train)

# Fast cleaning aligned with Week 1 findings
df_viz <- df %>%
  mutate(
    across(where(is.character), ~na_if(trimws(.), "")),
    Title = str_extract(Name, "[A-Za-z]+(?=\\.)"),
    Title = case_when(
      Title %in% c("Mlle", "Ms") ~ "Miss",
      Title == "Mme" ~ "Mrs",
      Title %in% c("Don", "Sir", "Jonkheer", "Rev", "Dr", "Col", "Major", "Capt", "Lady", "Countess", "Dona") ~ "Rare",
      TRUE ~ Title
    )
  )

# Group-median imputation for Age
age_medians <- df_viz %>% group_by(Title) %>% summarise(med = median(Age, na.rm = TRUE))
df_viz <- df_viz %>%
  left_join(age_medians, by = "Title") %>%
  mutate(Age = if_else(is.na(Age), med, Age)) %>%
  select(-med)
df_viz$Age[is.na(df_viz$Age)] <- median(df_viz$Age, na.rm = TRUE)

# Mode imputation for Embarked
df_viz$Embarked[is.na(df_viz$Embarked)] <- "S"

# Capping Fare at 1.5 * IQR to prevent visualization distortion
q25_fare <- quantile(df_viz$Fare, 0.25, na.rm = TRUE)
q75_fare <- quantile(df_viz$Fare, 0.75, na.rm = TRUE)
cap_fare <- q75_fare + 1.5 * (q75_fare - q25_fare)
df_viz <- df_viz %>% mutate(Fare_Capped = pmin(Fare, cap_fare))

# Factor formatting for labels
df_viz <- df_viz %>%
  mutate(
    Survived_Label = factor(Survived, levels = c(0, 1), labels = c("Perished", "Survived")),
    Pclass_Label = factor(Pclass, levels = c(1, 2, 3), labels = c("1st Class", "2nd Class", "3rd Class")),
    Embarked_Label = recode(Embarked, "C" = "Cherbourg", "Q" = "Queenstown", "S" = "Southampton"),
    Age_Group = cut(Age, breaks = c(0, 12, 18, 35, 60, 100), 
                    labels = c("Child (0-12)", "Teen (13-18)", "Young Adult (19-35)", "Middle-Aged (36-60)", "Senior (60+)"))
  )

# ------------------------------------------------------------------------------
# Visualization 1: Stacked 100% Proportion Bar Chart (Interaction Effect)
# ------------------------------------------------------------------------------
viz1 <- ggplot(df_viz, aes(x = Pclass_Label, fill = Survived_Label)) +
  geom_bar(position = "fill", width = 0.65, color = "white", linewidth = 0.5) +
  facet_wrap(~Sex) +
  scale_y_continuous(labels = scales::percent_format()) +
  scale_fill_manual(values = c("Perished" = "#C0392B", "Survived" = "#27AE60")) +
  theme_minimal(base_size = 12) +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    strip.text = element_text(face = "bold", size = 12),
    legend.position = "top"
  ) +
  labs(
    title = "Figure 1: Survival Proportions across Passenger Class and Sex",
    subtitle = "Demonstrates severe structural disadvantage for 3rd class and male passengers",
    x = "Passenger Class",
    y = "Proportion of Passengers",
    fill = "Outcome"
  )

ggsave("outputs/figures/fig1_bar_class_sex_survival.png", viz1, width = 7.5, height = 4.5, dpi = 300)

# ------------------------------------------------------------------------------
# Visualization 2: Violin & Boxplot Distribution Plot (Age & Class Dispersal)
# ------------------------------------------------------------------------------
viz2 <- ggplot(df_viz, aes(x = Pclass_Label, y = Age, fill = Survived_Label)) +
  geom_violin(position = position_dodge(0.8), alpha = 0.5, trim = FALSE) +
  geom_boxplot(position = position_dodge(0.8), width = 0.2, color = "black", outlier.shape = 21, outlier.size = 1.5) +
  scale_fill_manual(values = c("Perished" = "#E67E22", "Survived" = "#2980B9")) +
  theme_minimal(base_size = 12) +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    legend.position = "bottom"
  ) +
  labs(
    title = "Figure 2: Age Distribution Dynamics by Passenger Class and Survival",
    subtitle = "Violin contours display density; embedded boxplots highlight median and IQR",
    x = "Passenger Class",
    y = "Age (Years)",
    fill = "Outcome"
  )

ggsave("outputs/figures/fig2_violin_age_pclass.png", viz2, width = 8, height = 5, dpi = 300)

# ------------------------------------------------------------------------------
# Visualization 3: Trend Curve / Aggregated Line Chart (Age Trajectory)
# ------------------------------------------------------------------------------
age_trend <- df_viz %>%
  group_by(Age_Group) %>%
  summarise(
    Total = n(),
    Survivors = sum(Survived),
    Survival_Rate = Survivors / Total,
    .groups = "drop"
  )

viz3 <- ggplot(age_trend, aes(x = Age_Group, y = Survival_Rate, group = 1)) +
  geom_line(color = "#16A085", linewidth = 1.3) +
  geom_point(color = "#2C3E50", size = 4) +
  geom_text(aes(label = scales::percent(Survival_Rate, accuracy = 0.1)), vjust = -1.2, fontface = "bold", size = 3.8) +
  scale_y_continuous(labels = scales::percent_format(), limits = c(0, 0.7)) +
  theme_minimal(base_size = 12) +
  theme(plot.title = element_text(face = "bold", size = 14)) +
  labs(
    title = "Figure 3: Survival Probability Trajectory Across Age Cohorts",
    subtitle = "Children (<12) had high survival probability; sharpest mortality hit middle-aged and seniors",
    x = "Age Cohort",
    y = "Survival Rate (%)"
  )

ggsave("outputs/figures/fig3_line_age_trajectory.png", viz3, width = 7.5, height = 4.5, dpi = 300)

# ------------------------------------------------------------------------------
# Visualization 4: Scatter Plot with Loess Trendlines (Fare vs Age)
# ------------------------------------------------------------------------------
viz4 <- ggplot(df_viz, aes(x = Age, y = Fare_Capped, color = Survived_Label)) +
  geom_point(alpha = 0.6, size = 2) +
  geom_smooth(method = "loess", se = FALSE, linewidth = 1.2) +
  scale_color_manual(values = c("Perished" = "#E74C3C", "Survived" = "#3498DB")) +
  theme_minimal(base_size = 12) +
  theme(plot.title = element_text(face = "bold", size = 14), legend.position = "top") +
  labs(
    title = "Figure 4: Scatter Dispersion of Fare vs. Age with Loess Smoothing",
    subtitle = "Higher fares (> $40) correlate with higher density of survival across age bands",
    x = "Age (Years)",
    y = "Ticket Fare ($ - Capped at 1.5x IQR)",
    color = "Outcome"
  )

ggsave("outputs/figures/fig4_scatter_fare_age.png", viz4, width = 8, height = 5, dpi = 300)

# ------------------------------------------------------------------------------
# Visualization 5: Trellis / Lattice Multi-Panel Density (Demonstrating Lattice)
# ------------------------------------------------------------------------------
png("outputs/figures/fig5_lattice_fare_density.png", width = 800, height = 500, res = 120)
print(
  densityplot(~Fare_Capped | Embarked_Label, data = df_viz,
              groups = Survived_Label,
              auto.key = list(columns = 2, points = FALSE, lines = TRUE),
              main = "Figure 5: Lattice Trellis Density of Fare by Embarkation Port",
              xlab = "Fare ($ Capped)",
              ylab = "Density",
              plot.points = FALSE,
              lwd = 2)
)
dev.off()

cat("\n[SUCCESS] All 5 Week-2 visualizations exported to 'outputs/figures/' successfully!\n")