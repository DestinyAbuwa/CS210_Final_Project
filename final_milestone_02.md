# CS210 Final Project
# Destiny Abuwa

```
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <list>
#include <vector>
#include <memory>
#include <random>     // For random replacement
#include <limits>     // For frequency comparisons

using namespace std;

// ------------------- CityKey Struct and Hash --------------------

// A structure to hold the country code and city name as the key to cache city populations
struct CityKey {
    string countryCode;
    string cityName;

    // Overload equality operator to compare CityKey objects
    bool operator==(const CityKey& other) const {
        return cityName == other.cityName && countryCode == other.countryCode;
    }
};

// Hash function for CityKey, required for using CityKey as a key in unordered_map
struct CityKeyHash {
    size_t operator()(const CityKey& key) const {
        return hash<string>()(key.cityName) ^ hash<string>()(key.countryCode); // Combine hashes of cityName and countryCode
    }
};
