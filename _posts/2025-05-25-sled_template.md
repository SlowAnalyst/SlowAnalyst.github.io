---
layout: post
title: "Rust sled template"
summary: "러스트 임베디드 데이터베이스 sled 템플릿"
author: eveheeero
date: '2025-05-25 04:26:00 +0900'
category: ['etc', 'rust']
tags: rust
thumbnail: /assets/img/posts/default.png
keywords: rust, sled, template
usemathjax: false
permalink: /blog/rust-sled-template/
---


## 배경

사이드 프로젝트 중 sled 관련 공통 모듈을 만들어 공유하게 되었습니다.

## 요약

- `str` 및 `Uuid`를 통해 키 이용 가능

## 소스 코드

```rust
// serde = { version = "1.0.152", features = ["derive"] }
// serde_json = "1.0.91"
// sled = "0.34.7"
// flate2 = "1.1.1"
// uuid = { version = "1.17.0", features = ["fast-rng", "v4"] 
use serde::{Serialize, de::DeserializeOwned};
use sled::IVec;
use std::{io::Write, sync::LazyLock};
use uuid::Uuid;

static DB: LazyLock<sled::Db> = LazyLock::new(|| sled::open("db").unwrap());

pub struct DbRoot;

pub struct DbTree {
    pub inner: sled::Tree,
}

pub trait DbReturnTrait: DbGet + DbRaw {
    fn set<K: TreeKey, I: Serialize + ?Sized, R: DeserializeOwned>(
        &self,
        key: K,
        value: &I,
    ) -> Result<Option<R>, sled::Error> {
        let result = self.set_raw(key, value)?;
        Ok(result.map(|x| inflate_deserialize(&x)))
    }
    fn remove<K: TreeKey, R: DeserializeOwned>(&self, key: K) -> Result<Option<R>, sled::Error> {
        let result = self.remove_raw(key)?;
        Ok(result.map(|x| inflate_deserialize(&x)))
    }
}
pub trait DbTrait: DbGet + DbRaw {
    fn set<K: TreeKey, I: Serialize + ?Sized>(
        &self,
        key: K,
        value: &I,
    ) -> Result<Option<()>, sled::Error> {
        let result = self.set_raw(key, value)?;
        Ok(result.map(|_| ()))
    }
    fn new<I: Serialize + ?Sized>(&self, value: &I) -> Result<Uuid, sled::Error> {
        let key = Uuid::new_v4();
        self.set(key, value)?;
        Ok(key)
    }
    fn remove<K: TreeKey>(&self, key: K) -> Result<Option<()>, sled::Error> {
        let result = self.remove_raw(key)?;
        Ok(result.map(|_| ()))
    }
}
pub trait DbRaw {
    fn set_raw<K: TreeKey, I: Serialize + ?Sized>(
        &self,
        key: K,
        value: &I,
    ) -> Result<Option<IVec>, sled::Error>;
    fn remove_raw<K: TreeKey>(&self, key: K) -> Result<Option<IVec>, sled::Error>;
}
pub trait DbGet {
    fn get<K: TreeKey, R: DeserializeOwned>(&self, key: K) -> Result<Option<R>, sled::Error>;
}

impl DbRoot {
    pub fn get_tree<K: TreeKey>(&self, name: K) -> DbTree {
        DbTree {
            inner: DB.open_tree(name.key()).unwrap(),
        }
    }
    pub fn drop_tree<K: TreeKey>(&self, name: K) -> Result<bool, sled::Error> {
        DB.drop_tree(name.key())
    }
    pub fn flush(&self) -> Result<(), sled::Error> {
        DB.flush().map(|_| ())
    }
}

impl DbRaw for DbRoot {
    fn set_raw<K: TreeKey, I: Serialize + ?Sized>(
        &self,
        key: K,
        value: &I,
    ) -> Result<Option<IVec>, sled::Error> {
        let compressed_data = deflate_serialize(value);
        DB.insert(key.key(), compressed_data)
    }
    fn remove_raw<K: TreeKey>(&self, key: K) -> Result<Option<IVec>, sled::Error> {
        DB.remove(key.key())
    }
}
impl DbGet for DbRoot {
    fn get<K: TreeKey, R: DeserializeOwned>(&self, key: K) -> Result<Option<R>, sled::Error> {
        DB.get(key.key()).map(|x| {
            x.map(|v| inflate(&v))
                .map(|x| serde_json::from_slice(&x).unwrap())
        })
    }
}

impl DbTree {
    pub fn name(&self) -> String {
        str::from_utf8(&self.inner.name()).unwrap().to_string()
    }
    pub fn flush(&self) -> Result<(), sled::Error> {
        self.inner.flush().map(|_| ())
    }
}
impl DbRaw for DbTree {
    fn set_raw<K: TreeKey, I: Serialize + ?Sized>(
        &self,
        key: K,
        value: &I,
    ) -> Result<Option<IVec>, sled::Error> {
        let compressed_data = deflate_serialize(value);
        self.inner.insert(key.key(), compressed_data)
    }
    fn remove_raw<K: TreeKey>(&self, key: K) -> Result<Option<IVec>, sled::Error> {
        self.inner.remove(key.key())
    }
}
impl DbGet for DbTree {
    fn get<K: TreeKey, R: DeserializeOwned>(&self, key: K) -> Result<Option<R>, sled::Error> {
        self.inner.get(key.key()).map(|x| {
            x.map(|v| inflate(&v))
                .map(|x| serde_json::from_slice(&x).unwrap())
        })
    }
}
impl DbTrait for DbRoot {}
impl DbReturnTrait for DbRoot {}
impl DbTrait for DbTree {}
impl DbReturnTrait for DbTree {}

fn inflate_deserialize<T: DeserializeOwned>(data: &[u8]) -> T {
    let decompressed_data = inflate(data);
    serde_json::from_slice(&decompressed_data).unwrap()
}
fn deflate_serialize<T: Serialize + ?Sized>(value: &T) -> Vec<u8> {
    let data = serde_json::to_vec(value).unwrap();
    deflate(&data)
}
fn deflate(data: &[u8]) -> Vec<u8> {
    use flate2::Compression;
    use flate2::write::DeflateEncoder;
    let mut encoder = DeflateEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(data).unwrap();
    encoder.finish().unwrap()
}

fn inflate(data: &[u8]) -> Vec<u8> {
    use flate2::read::DeflateDecoder;
    use std::io::Read;
    let mut decoder = DeflateDecoder::new(data);
    let mut out = Vec::new();
    decoder.read_to_end(&mut out).unwrap();
    out
}

#[test]
fn duplicate_test() {
    let _ = DbTrait::set(&DbRoot, "tree", "asdf").unwrap();
    let _ = DbRoot.get_tree("tree");
    let _ = DbRoot.get_tree("tree");
}

#[test]
fn duplicate_drop() {
    let _ = DbRoot.drop_tree("tree");
    let _ = DbTrait::remove(&DbRoot, "tree");
}

#[test]
fn inspect() {
    println!("Master: {}", str::from_utf8(&DB.name()).unwrap());
    DB.iter().for_each(|item| {
        if let Ok((key, value)) = item {
            println!(
                "Key: {}, Value: {}",
                str::from_utf8(&key).unwrap(),
                str::from_utf8(&inflate(&value)).unwrap()
            );
        }
    });
    let tree_names = DB.tree_names();
    tree_names.iter().for_each(|name| {
        println!("Tree: {}", str::from_utf8(name).unwrap());
        if let Ok(tree) = DB.open_tree(name) {
            tree.iter().for_each(|item| {
                if let Ok((key, value)) = item {
                    println!(
                        "  Key: {}, Value: {}",
                        str::from_utf8(&key).unwrap(),
                        str::from_utf8(&inflate(&value)).unwrap()
                    );
                }
            });
        }
        println!();
    });
}

pub trait TreeKey {
    fn key(&self) -> impl AsRef<[u8]>;
}
impl TreeKey for &str {
    fn key(&self) -> impl AsRef<[u8]> {
        self
    }
}
impl TreeKey for String {
    fn key(&self) -> impl AsRef<[u8]> {
        self
    }
}
impl TreeKey for Uuid {
    fn key(&self) -> impl AsRef<[u8]> {
        self.to_string()
    }
}
```

## 작성자의 글

- .

## 참조

- .
