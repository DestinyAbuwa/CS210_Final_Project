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

    // Add or update a value in the cache
    void put(const CityKey& key, const string& population) {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            // If key already exists, remove old entry
            cacheList.erase(it->second);
            cacheMap.erase(it);
        }
        // Insert new item at the front of the list
        cacheList.push_front({key, population});
        // Update the map to point to the new list position
        cacheMap[key] = cacheList.begin();

        // Remove least recently used item if we go over size
        if (cacheList.size() > maxSize) {
            auto last = cacheList.end();
            last--;
            cacheMap.erase(last->first);  // Remove from map
            cacheList.pop_back();         // Remove from list
        }
    }

};


// Search the CSV file line-by-line for a matching city
bool searchCSV(const CityKey& key, string& population) {
    ifstream file("world_cities.csv");
    if (!file.is_open()) return false;  // Couldn't open file

    string line;
    getline(file, line); // Skip the header line
    while (getline(file, line)) {
        stringstream ss(line);
        string country, city, pop;

        // Parse CSV columns
        getline(ss, country, ',');
        getline(ss, city, ',');
        getline(ss, pop, ',');

        // Match input key with line from CSV
        if (key.countryCode == country && key.cityName == city) {
            population = pop;
            return true;  // Match found
        }
    }
    return false;  // Not found
}




