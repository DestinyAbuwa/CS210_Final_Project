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
#include <random>
#include <limits>
#include <map>

using namespace std;

// ------------------- CityKey Struct and Hash --------------------
// Struct representing a city-country pair uniquely.
struct CityKey {
    string countryCode;
    string cityName;

    // Equality operator used by hash map
    bool operator==(const CityKey& other) const {
        return cityName == other.cityName && countryCode == other.countryCode;
    }
};

// Custom hash function for CityKey (used in unordered_map)
struct CityKeyHash {
    size_t operator()(const CityKey& key) const {
        return hash<string>()(key.cityName) ^ hash<string>()(key.countryCode);
    }
};
