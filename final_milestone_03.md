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

// ------------------- Cache Interface --------------------
// Abstract base class for all cache strategies.
class CityCache {
public:
    virtual bool get(const CityKey& key, string& population) = 0;
    virtual void put(const CityKey& key, const string& population) = 0;
    virtual ~CityCache() {}
};

// ------------------- LFU Cache --------------------
// Least Frequently Used Cache
class LFUCache : public CityCache {
private:
    struct CacheEntry {
        string population;       // Value
        int frequency;           // How often it's used
        list<CityKey>::iterator orderIt; // For order tracking
    };

    unordered_map<CityKey, CacheEntry, CityKeyHash> cacheMap;
    list<CityKey> orderList; // Maintains insertion order
    const size_t maxSize = 10;

public:
    bool get(const CityKey& key, string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            it->second.frequency++; // Update usage frequency
            population = it->second.population;
            return true;
        }
        return false;
    }

    void put(const CityKey& key, const string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            it->second.population = population;
            it->second.frequency++;
            return;
        }

        // Evict least frequently used if cache is full
        if (cacheMap.size() >= maxSize) {
            int minFreq = numeric_limits<int>::max();
            CityKey toRemove;
            for (const auto& k : orderList) {
                int freq = cacheMap[k].frequency;
                if (freq < minFreq) {
                    minFreq = freq;
                    toRemove = k;
                }
            }
            orderList.remove(toRemove);
            cacheMap.erase(toRemove);
        }

        orderList.push_back(key);
        cacheMap[key] = {population, 1, --orderList.end()};
    }
};

// ------------------- FIFO Cache --------------------
// First-In, First-Out Cache
class FIFOCache : public CityCache {
private:
    list<pair<CityKey, string>> cacheList;
    unordered_map<CityKey, list<pair<CityKey, string>>::iterator, CityKeyHash> cacheMap;
    const size_t maxSize = 10;

public:
    bool get(const CityKey& key, string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            population = it->second->second;
            return true;
        }
        return false;
    }

    void put(const CityKey& key, const string& population) override {
        // Remove existing entry if found
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            cacheList.erase(it->second);
            cacheMap.erase(it);
        }

        // Evict oldest if full
        if (cacheList.size() >= maxSize) {
            auto oldest = cacheList.front();
            cacheMap.erase(oldest.first);
            cacheList.pop_front();
        }

        cacheList.push_back({key, population});
        cacheMap[key] = --cacheList.end();
    }
};

// ------------------- Random Cache --------------------
// Random Replacement Cache
class RandomCache : public CityCache {
private:
    unordered_map<CityKey, string, CityKeyHash> cacheMap;
    vector<CityKey> keys; // To select random element
    const size_t maxSize = 10;
    random_device rd;
    mt19937 gen;

public:
    RandomCache() : gen(rd()) {}

    bool get(const CityKey& key, string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            population = it->second;
            return true;
        }
        return false;
    }

    void put(const CityKey& key, const string& population) override {
        if (cacheMap.find(key) != cacheMap.end()) {
            cacheMap[key] = population;
            return;
        }

        // Evict a random entry if full
        if (cacheMap.size() >= maxSize) {
            uniform_int_distribution<> dis(0, keys.size() - 1);
            int idx = dis(gen);
            CityKey toRemove = keys[idx];
            cacheMap.erase(toRemove);
            keys.erase(keys.begin() + idx);
        }

        keys.push_back(key);
        cacheMap[key] = population;
    }
};

// ------------------- Trie Node Definition --------------------
// Each node represents a letter and stores children and country-population map
struct TrieNode {
    unordered_map<char, TrieNode*> children;
    map<string, string> countryPopMap; // countryCode -> population
};

// Trie to store cities efficiently
class CityTrie {
private:
    TrieNode* root;

public:
    CityTrie() {
        root = new TrieNode();
    }

    // Insert a city-country-population into the trie
    void insert(const string& city, const string& country, const string& population) {
        TrieNode* node = root;
        for (char ch : city) {
            if (!node->children[ch])
                node->children[ch] = new TrieNode();
            node = node->children[ch];
        }
        node->countryPopMap[country] = population;
    }

    // Search the trie for a given city and country
    bool search(const string& city, const string& country, string& population) {
        TrieNode* node = root;
        for (char ch : city) {
            if (!node->children.count(ch))
                return false;
            node = node->children[ch];
        }
        auto it = node->countryPopMap.find(country);
        if (it != node->countryPopMap.end()) {
            population = it->second;
            return true;
        }
        return false;
    }

    // Cleanup all nodes
    ~CityTrie() {
        clear(root);
    }

    void clear(TrieNode* node) {
        for (auto& pair : node->children)
            clear(pair.second);
        delete node;
    }
};


// ------------------- Load Trie From CSV --------------------
// Read the world_cities.csv file and load data into the trie
void loadCSVtoTrie(const string& filename, CityTrie& trie) {
    ifstream file(filename);
    if (!file.is_open()) {
        cerr << "Failed to open file: " << filename << endl;
        return;
    }

    string line;
    getline(file, line); // Skip header

    while (getline(file, line)) {
        stringstream ss(line);
        string country, city, population;
        getline(ss, country, ',');
        getline(ss, city, ',');
        getline(ss, population, ',');
        trie.insert(city, country, population); // Insert into trie
    }

    cout << "Loaded cities into Trie.\n";
}

int main() {
    unique_ptr<CityCache> cache; // Polymorphic cache
    CityTrie trie;
    int choice;

    loadCSVtoTrie("world_cities.csv", trie); // Load file once at start

    // Let user pick a cache strategy
    cout << "Select caching strategy:\n";
    cout << "1. LFU (Least Frequently Used)\n";
    cout << "2. FIFO (First-In, First-Out)\n";
    cout << "3. Random Replacement\n";
    cout << "Enter choice: ";
    cin >> choice;
    cin.ignore();

    // Create cache based on user choice
    if (choice == 1) cache = make_unique<LFUCache>();
    else if (choice == 2) cache = make_unique<FIFOCache>();
    else if (choice == 3) cache = make_unique<RandomCache>();
    else {
        cout << "Invalid choice.\n";
        return 1;
    }

    // Loop: allow users to search for city populations
    while (true) {
        string city, country;
        cout << "\nEnter city name (or 'exit'): ";
        getline(cin, city);
        if (city == "exit") break;

        cout << "Enter country code: ";
        getline(cin, country);

        CityKey key = {country, city};
        string population;

        // Try to find in cache
        if (cache->get(key, population)) {
            cout << "(Cache) Population of " << city << ", " << country << ": " << population << endl;
        }
        // Else fall back to trie
        else if (trie.search(city, country, population)) {
            cout << "(Trie) Population of " << city << ", " << country << ": " << population << endl;
            cache->put(key, population); // Save in cache
        } else {
            cout << "City not found.\n";
        }
    }

    return 0;
}
