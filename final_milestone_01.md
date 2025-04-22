# CS210 Final Project
# Destiny Abuwa

```
#include <iostream>          
#include <fstream>           
#include <sstream>           
#include <string>            
#include <unordered_map>     // For the hash map (fast lookups)
#include <list>              // For maintaining access order in the cache
using namespace std;

// Struct to represent a city by its country code and city name
struct CityKey {
    string countryCode;
    string cityName;

    // Define equality for CityKey to compare keys in unordered_map
    bool operator==(const CityKey& other) const {
        return cityName == other.cityName && countryCode == other.countryCode;
    }
};
