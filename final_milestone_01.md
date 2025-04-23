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

// Custom hash function for CityKey so we can use it in an unordered_map
struct CityKeyHash {
    size_t operator()(const CityKey& key) const {
        // XOR hash of city and country strings (basic combination method)
        return hash<string>()(key.cityName) ^ hash<string>()(key.countryCode);
    }
};

// Least Recently Used Cache class for storing recent city lookups
class CityCache {
private:
    // A list stores key-value pairs, with most recent at the front
    list<pair<CityKey, string>> cacheList;
    // Map from CityKey to list iterator for fast O(1) access
    unordered_map<CityKey, list<pair<CityKey, string>>::iterator, CityKeyHash> cacheMap;
    const size_t maxSize = 10;  // Only store 10 recent items

public:
    // Try to get a value from the cache
    bool get(const CityKey& key, string& population) {
        auto it = cacheMap.find(key);  // Check if key is in cache
        if (it != cacheMap.end()) {
            // Move found item to the front of the list (most recent)
            cacheList.splice(cacheList.begin(), cacheList, it->second);
            population = it->second->second;  // Get the population from the list
            return true;  // Cache hit
        }
        return false;  // Cache miss
    }

};






