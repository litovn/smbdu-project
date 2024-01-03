# Database Management Project
A course project on "Systems and Methods for Big and Unstructured Data" presented to work with a database management system for the 2023/2024 academic year.
The goal is to choose a database technology (Neo4j, MongoDB, or Elasticsearch) and a dataset (with at least 20,000 data points) for analysis that will be imported into the chosen database, performing various queries, and presenting the findings in a structured report. 

The following repository was created to store the complete outcomes of the queries presented in the report.

___ 

## Introduction
In the dynamic landscape of data management, the emergence of Big and Unstructured Data presents unprecedented opportunities, while significant challenges also arise in their management. As we navigate this immense sea of information, it becomes increasingly important to adopt innovative strategies to deploy systems capable of handling large volumes of diverse data while acquiring, generating, and processing at an accelerated pace.

The database technology known as [Elastic Stack v8.11](https://www.elastic.co/elastic-stack) provides open-source tools for data ingestion, enrichment, storage, analysis, and visualization, and consists of Elasticsearch, Logstash, Kibana and Beats.

It might seem trivial, but the importance of salmon extends beyond its role as a food source. It encompasses ecological, economic, cultural, and social dimensions, making it a species of significant value and concern for various stakeholders.
The conservation and sustainable management of salmon populations are crucial for maintaining the health of ecosystems and supporting the well-being of human communities. Knowing population demographics, growth rates and trends is particularly valuable to fisheries managers who must perform population assessments to inform management decisions. For this reason, analyzing a comprehensive dataset spanning several decades can offer a wealth of information that extends beyond immediate concerns. It has the potential to inform sustainable management practices, contribute to scientific knowledge, and support the long-term health and resilience of salmon ecosystems in Alaska.

## The Dataset
The original dataset under consideration, published in 2018 by Jeanette Clark of the "National Center for Ecological Analysis and Synthesis", along with Rich Brenner and Bert Lewis from the "Alaska Department of Fish and Game, Division of Commercial Fisheries", was sourced from the [Knowledge Network for Biocomplexity platform](https://knb.ecoinformatics.org/view/doi:10.5063/F1707ZTM).

The dataset gathers data compiled from an annual sampling of commercial and subsistence salmon harvests and research projects throughout Alaska from 1922 to 2017. It includes various data on five different salmon species.

## Data Wrangling
The original dataset composed of 14.347.552 data points and 27 columns, has been concluded to be extensive, including numerous variables that were not directly relevant to the research proposed in this paper, while also containing missing values and inconsistencies.
The following Python script was written to clean the data, enhancing its overall quality, ensuring that the results presented in the paper are based on accurate and trustworthy information, and streamlining the dataset to include only essential variables necessary for addressing the specific research objectives in mind.

```python
import pandas

data = pandas.read_csv('ASL_master.csv')
sub_data = data[['Species','sampleYear','sampleDate','ASLProjectType','DataSource','Gear','Length','Weight','Sex','Salt.Water.Age','Fresh.Water.Age','Lat','Lon','LocationID','SASAP.Region']].dropna()

sample = sub_data.sample(frac=(1))
sample.to_csv('ASL_sample.csv', index=False)
```

Because of this process of transformation, we managed to exclude rows that contained missing or suspicious values and select specific columns that will help with the analysis by exporting a subset of the original set that will now include 112.847 data points and 15 columns.

## The Queries
### Q1. Data mapping
```
GET /salmon_dataset/_mapping
```
[Output Q1](Queries/Q01_retrievestruct.json)

### Q2. Chinook in Southeast region summer 2016
Retrieve all the various attributes regarding the salmon of the chinook species that were caught in the Southeast region in the summer season of the year 2016. 
```
GET /salmon_dataset/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "Species": "chinook"
          }
        },
        {
          "match": {
            "SASAP_Region": "Southeast"
          }
        },
        {
          "range": {
            "sampleDate": {
              "gte": "2016-06-21",
              "lte": "2016-09-22"
            }
          }
        }
      ]
    }
  }
} 
```
[Output Q2](Queries/Q02_southeastChinook_summer16.json)

### Q3. Total count of ASL Project types
Return the total count of samples for each Age, Sex, and Length (ASL) Project type.
```
GET /salmon_dataset/_search
{
  "aggs": {
    "project_type_count": {
      "terms": {
        "field": "ASLProjectType"
      }
    }
  },
  "size": 0
}
```
[Output Q3](Queries/Q03_countASLpTypes.json)

### Q4. Specific samples in location bounding box
Retrieve and count female salmon samples caught for subsistence, located within a specified geographic area, and group them by the type of gear used to catch the sample.
```
GET /salmon_dataset/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "Sex": "female"
          }
        },
        {
          "term": {
            "ASLProjectType": "subsistence catch"
          }
        },
        {
          "geo_bounding_box": {
            "location": {
              "top_left": {
                "lat": 65,
                "lon": -170
              },
              "bottom_right": {
                "lat": 55,
                "lon": -130
              }
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "gear_used": {
      "terms": {
        "field": "Gear"
      }
    }
  },
  "size": 0
}
```
[Output Q4](Queries/Q04_locationBB.json)

### Q5. Average weight of specific salmon
Find the average weight of female species of pink salmon grouped by the year the sample was retrieved, excluding samples with a length of less than 500 mm.
```
GET /salmon_dataset/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "Species": "pink"
          }
        },
        {
          "match": {
            "Sex": "female"
          }
        }
      ]
    }
  },
  "aggs": {
    "avg_weight_per_year": {
      "terms": {
        "field": "sampleYear"
      },
      "aggs": {
        "filtered_weight": {
          "filter": {
            "range": {
              "Length": {
                "gte": 500
              }
            }
          },
          "aggs": {
            "avg_weight": {
              "avg": {
                "field": "Weight"
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}
```
[Output Q5](Queries/Q05_salmon_avgWeight.json)

### Q6. Coho max, min and average length
Find the maximum, minimum and average lengths of the samples of coho salmon collected by each organization from which the sample was sourced, excluding "Bristol Bay - Neala Kendall".
```
GET /salmon_dataset/_search
{
  "query": {
    "match": {
      "Species": "coho"
    }
  },
  "aggs": {
    "avg_len_bysource": {
      "terms": {
        "field": "DataSource",
        "exclude": "Bristol Bay - Neala Kendall"
      },
      "aggs": {
        "avg_length": {
          "avg": {
            "field": "Length"
          }
        },
        "min_length": {
          "min": {
            "field": "Length"
          }
        },
        "max_length": {
          "max": {
            "field": "Length"
          }
        }
      }
    }
  },
  "size": 0
}
```
[Output Q6](Queries/Q06_cohoMaxMinAvgLen.json)

### Q7. Bristol Bay sockeye salmon analysis and insights
Retrieve information regarding the sockeye salmon samples in the Bristol Bay region, focusing on samples with a saltwater age of less than two and a freshwater age greater than or equal to one. 
Provide details on yearly trends, analyzing the average weight, length and most common caught sex and gear used (excluding intervals with zero samples), grouping by location and ASL project type.
```
GET /salmon_dataset/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "SASAP_Region": "Bristol Bay"
          }
        },
        {
          "term": {
            "Species": "sockeye"
          }
        },
        {
          "range": {
            "Salt_Water_Age": {
              "lt": 2
            }
          }
        },
        {
          "range": {
            "Fresh_Water_Age": {
              "gte": 1
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "groupby_location": {
      "terms": {
        "field": "LocationID"
      },
      "aggs": {
        "tot_samples": {
          "cardinality": {
            "field": "@timestamp"
          }
        },
        "groupby_ASLp": {
          "terms": {
            "field": "ASLProjectType"
          },
          "aggs": {
            "yearly_trend": {
              "date_histogram": {
                "field": "sampleDate",
                "calendar_interval": "year",
                "format": "yyyy",
                "min_doc_count": 1
              },
              "aggs": {
                "avg_weight_yearly": {
                  "avg": {
                    "field": "Weight"
                  }
                },
                "avg_length_yearly": {
                  "avg": {
                    "field": "Length"
                  }
                },
                "common_sex": {
                  "terms": {
                    "field": "Sex",
                    "size": 1
                  }
                },
                "common_gear": {
                  "terms": {
                    "field": "Gear",
                    "size": 1
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "size": 0
} 
```
[Output Q7](Queries/Q07_bristolSockeye_insight.json)

### Q8. Analysis of monthly salmon samples
Provide an analysis with both temporal patterns and the monthly seasonal effect on the length of salmon samples.
For each species, include information on the sample across different months, considering their overall age, alongside the average length for each month-species-age combination.
```
GET /salmon_dataset/_search
{
  "aggs": {
    "species_type": {
      "terms": {
        "field": "Species"
      },
      "aggs": {
        "monthly_distribution": {
          "date_histogram": {
            "field": "sampleDate",
            "calendar_interval": "month",
            "format": "MMM-yyyy",
            "min_doc_count": 1
          },
          "aggs": {
            "different_days": {
              "cardinality": {
                "field": "sampleDate"
              }
            },
            "sample_dates": {
              "terms": {
                "field": "sampleDate",
                "format": "dd"
              },
              "aggs": {
                "tot_age": {
                  "terms": {
                    "field": "Age"
                  },
                  "aggs": {
                    "avg_length": {
                      "avg": {
                        "field": "Length"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}
```
[Output Q8](Queries/Q08_monthly_analysis.json)

### Q9. Characteristics by weight ranges
Analyze the dataset by dividing it into weight ranges and then providing statistics on location distribution, species distribution within locations, and their sex.
For each sex, give some generic info and determine the biggest younger fish and smallest oldest fish.
```
GET /salmon_dataset/_search
{
  "size": 0,
  "aggs": {
    "weight_ranges": {
      "range": {
        "field": "Weight",
        "ranges": [
          {
            "from": 500,
            "to": 700
          },
          {
            "from": 700,
            "to": 1000
          },
          {
            "from": 1000
          }
        ]
      },
      "aggs": {
        "location_distribution": {
          "terms": {
            "field": "LocationID"
          },
          "aggs": {
            "species_distribution": {
              "terms": {
                "field": "Species"
              },
              "aggs": {
                "sex_distribution": {
                  "terms": {
                    "field": "Sex"
                  },
                  "aggs": {
                    "stats": {
                      "stats": {
                        "field": "Length"
                      }
                    },
                    "big_youngest": {
                      "top_hits": {
                        "_source": [
                          "Age",
                          "Length"
                        ],
                        "size": 1,
                        "sort": [
                          {
                            "Age": {
                              "order": "asc"
                            }
                          },
                          {
                            "Length": {
                              "order": "desc"
                            }
                          }
                        ]
                      }
                    },
                    "small_oldest": {
                      "top_hits": {
                        "_source": [
                          "Age",
                          "Length"
                        ],
                        "size": 1,
                        "sort": [
                          {
                            "Age": {
                              "order": "desc"
                            }
                          },
                          {
                            "Length": {
                              "order": "asc"
                            }
                          }
                        ]
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
[Output Q9](Queries/Q09_charachteristics_byWeightRange.json)

### Q10. Comparative analysis of the demographics across time
Analyze and compare salmon demographics between seasons of the 1970-'75 and 2010-'15 time periods.
```
GET /salmon_dataset/_search
{
  "size": 0,
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "sampleYear": {
              "gte": "1970",
              "lte": "1975"
            }
          }
        },
        {
          "range": {
            "sampleYear": {
              "gte": "2010",
              "lte": "2015"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "time_comparison": {
      "terms": {
        "script": {
          "source": """
                        def year = doc['sampleDate'].value.getYear();
                        if (year >= 1970 && year <= 1975) return '1970-1975';
                        else if (year >= 2010 && year <= 2015) return '2010-2015';
                      """
        },
        "size": 2
      },
      "aggs": {
        "seasonal_distribution": {
          "terms": {
            "script": {
              "source": """
                            def month = doc['sampleDate'].value.getMonthValue();
                            if (month >= 3 && month <= 5) return 'Spring';
                            else if (month >= 6 && month <= 8) return 'Summer';
                            else if (month >= 9 && month <= 11) return 'Fall';
                            else return 'Winter';
                          """
            },
            "size": 4
          },
          "aggs": {
            "species_terms": {
              "terms": {
                "field": "Species"
              },
              "aggs": {
                "sex_distribution": {
                  "terms": {
                    "field": "Sex"
                  }
                },
                "age_distribution": {
                  "histogram": {
                    "field": "Age",
                    "interval": 1
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
[Output Q10](Queries/Q10_demographicAnalsysi_timeranges.json)
