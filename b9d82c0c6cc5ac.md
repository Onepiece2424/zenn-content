---
title: "Google Mapsでドラッグ可能なピンと情報ウィンドウを表示する方法"
emoji: "🫠"
type: "tech"
topics:
  - "rails"
  - "react"
  - "ruby"
  - "typescript"
  - "googlemap"
published: true
published_at: "2024-09-14 14:48"
---

### はじめに

この記事では、Typescript,React,Rails,Google Maps APIを使用して、ドラッグ可能なピン（= マーカー）と情報ウィンドウを実装する方法を解説します。

具体的には、マーカーを地図上でドラッグ可能にし、クリック時にその位置情報（緯度・経度）や住所データを表示する情報ウィンドウを表示する機能を作成します。

アプリケーションは、フロント側はTypescript,React、バックエンドはRailsをメインに用いて実装しました。

よかったら参考にしていただけますと幸いです。

<br>

### デモ画像
![](https://storage.googleapis.com/zenn-user-upload/417c6ce4e90f-20240914.png)


<br>

実装手順は下記の通りです。

1. 環境変数の設定
2. マップとマーカーを表示するためのコンポーネントの作成
3. マップの作成
4. マーカーの作成
5. 情報ウィンドウの作成
6. 住所情報を取得

次に1つずつどのようなことをおこなったか説明していきます。

<br>

### 環境変数の設定

まず、Google Maps APIを使用するために、Google Cloud PlatformでAPIキーを取得し、`.env`ファイルに追加しておきます。

Google Maps APIを使用するには、Google Cloud Platform (GCP) でプロジェクトを作成し、APIキーを取得する必要があります。

↓こちらのドキュメントを参考にしてください。

https://developers.google.com/maps/documentation/javascript/get-api-key?hl=ja

ざっくり説明すると、APIキーを取得するには以下の手順を行います。

1. **GCPにログイン**: Google Cloud Consoleにログインします。
2. **プロジェクトの作成**: 左上のメニューから「プロジェクトを作成」を選び、新しいプロジェクトを作成します。
3. **APIとサービス**: 左側のナビゲーションメニューから「APIとサービス」→「ダッシュボード」に移動します。
4. **APIの有効化**: 「APIとサービスを有効化」ボタンをクリックし、「Google Maps JavaScript API」を検索して有効化します。
5. **APIキーの取得**: 「認証情報」タブから「APIキーを作成」ボタンをクリックしてAPIキーを生成します。

次に、`.env`ファイルに先ほど作成したAPIキーを設定します。

このファイルは、Reactアプリケーションのルートディレクトリに配置してください。

```jsx
// 先ほど作成したAPIキーを設定（your_google_maps_api_key_hereはAPIキーの値です）
REACT_APP_GOOGLE_MAP_API_KEY=your_google_maps_api_key_here
```

ここでは２点注意点があります。

- **変数名のプレフィックス**

Reactアプリケーションでは、環境変数を`REACT_APP_`で始める必要があります。

これにより、環境変数がアプリケーション内で利用可能になります。

- **APIキーの保護**

`.env`ファイルは、プロジェクトのルートディレクトリに置くことで、ビルド時に環境変数が読み込まれます。

ただし、このファイルは公開リポジトリには含めないようにし、`.gitignore`ファイルに追加しておくことが推奨されます。

https://www.atlassian.com/ja/git/tutorials/saving-changes/gitignore

環境変数の設定が終わったら、console.logを使用し、一度 **.envファイルに設定したAPIキーの値が取得できるかどうか確認すること**をお勧めします。

<br>

### 地図とマーカーを表示するためのコンポーネントの作成

次に、地図とマーカーを表示するためのコンポーネントを作成します。

以下のコードでは、地図の中心とピンの初期位置を設定し、マーカーがドラッグされたときに新しい位置を更新するイベントハンドラを設定しています。

```jsx
import { useState } from 'react';
import { Wrapper, Status } from "@googlemaps/react-wrapper";
import Maps from './Maps';
import Marker from './Marker';

type GoogleMapsProps = {
  lat: number;
  lng: number;
};

const GoogleMaps = () => {
  const [lat, setLat] = useState<number>(35.7140371);
  const [lng, setLng] = useState<number>(139.7925173);

  const render = (status: Status) => {
    return <h1>{status}</h1>;
  };

  const apiKey = process.env.REACT_APP_GOOGLE_MAP_API_KEY as string;

  const position: GoogleMapsProps = {
    lat: lat as number,
    lng: lng as number
  };

  const handleMarkerDragEnd = (e: google.maps.MapMouseEvent) => {
    if (e.latLng) {
      setLat(e.latLng.lat());
      setLng(e.latLng.lng());
    }
  };

  return (
    <Wrapper apiKey={apiKey} render={render}>
      <div className='main-container'>
        <Maps
          style={{ maxWidth: '800px', aspectRatio: '16 / 9', margin: '10px auto' }}
          center={position}
        >
          <Marker
            position={position}
            draggable={true}
            onDragEnd={handleMarkerDragEnd}
          />
        </Maps>
      </div>
    </Wrapper>
  );
}

export default GoogleMaps;
```

実装時は下記の記事を参考にしました。

https://developers.google.com/maps/documentation/javascript/react-map?hl=ja

ここでの重要な点は大きく2つです。

1. Wrapper,Maps,Marker コンポーネントの各配置場所とデフォルトの座標（緯度・経度）をpropsとして渡すこと
2. handleMarkerDragEnd 関数というマーカーがドラッグされている最中やドラッグ終了時のイベント処理を作成すること

1.は先ほど[添付したドキュメント](https://developers.google.com/maps/documentation/javascript/react-map?hl=ja#marker-child-component)を参考にしていただけると幸いです。

2.は少し解説します。`handleMarkerDragEnd` 関数で、マーカーがドラッグされている最中やドラッグ終了時のイベントを処理します。

```jsx
  // マーカーをドラッグ時に実行
  const handleMarkerDragEnd = (e: google.maps.MapMouseEvent) => {
    if (e.latLng) {
      setLat(e.latLng.lat());
      setLng(e.latLng.lng());
    }
  };
```

`e.latLng` が存在する場合、マーカーの新しい位置を取得し、状態 (`lat` と `lng`) を更新します。

これにより、ユーザーがマーカーを動かすとその新しい位置に応じて地図が更新されます。

<br>

### マップの作成

次は、Google Mapsを表示するコンポーネントの作成です。

マップのクリックイベントを処理するために、`onClick`プロパティを追加しています。

```jsx
import React, { useState, useRef, useEffect, ReactElement } from "react";

type MapProps = google.maps.MapOptions & {
  className?: string;
  style?: React.CSSProperties;
  children?: React.ReactNode;
  onClick?: (e: google.maps.MapMouseEvent) => void; // クリックイベントを追加
};

const Maps = ({ children, className, style, onClick, ...options }: MapProps) => {
  const ref = useRef<HTMLDivElement>(null);
  const [map, setMap] = useState<google.maps.Map>();

  useEffect(() => {
    if (ref.current && !map) {
      const mapInstance = new window.google.maps.Map(ref.current, {
        center: options.center,
        zoom: options.zoom || 16,
      });

      if (onClick) {
        mapInstance.addListener("click", onClick); // クリックイベントをマップに追加
      }

      setMap(mapInstance);
    }
  }, [ref, map, options.center, options.zoom, onClick]);

  useEffect(() => {
    if (map && options.center) {
      map.setCenter(options.center);
    }
  }, [map, options.center]);

  return (
    <div ref={ref} className={className} style={style}>
      {React.Children.map(children, (child) => {
        if (React.isValidElement(child)) {
          return React.cloneElement(child as ReactElement<any>, { map });
        }
        return child;
      })}
    </div>
  );
};

export default Maps;
```

ここでの重要な点は大きく2つです。

1. 地図の初期化
2. 地図の中心位置の更新

<br>
1. 地図の初期化 は、下記の部分で行なっています。

```jsx
useEffect(() => {
  if (ref.current && !map) {
    const mapInstance = new window.google.maps.Map(ref.current, {
      center: options.center,
      zoom: options.zoom || 16,
    });

    if (onClick) {
      mapInstance.addListener("click", onClick); // クリックイベントをマップに追加
    }

    setMap(mapInstance);
  }
}, [ref, map, options.center, options.zoom, onClick]);
```

上記のコードでは、まず、`useEffect` フックを使用して、コンポーネントが初めてレンダリングされたときに地図を初期化しています。

次に、`ref.current` の有無を確認し、まだ地図が初期化されていない場合に、Google Mapsのインスタンスを作成しています。

`options.center` で地図の中心を設定し、`options.zoom` でズームレベルを設定しています（デフォルトは 16 です）。

`onClick` が指定されている場合、地図にクリックイベントリスナーを追加しています。

最後に、地図のインスタンスを状態 (`map`) に保存します。

<br>
2. 地図の中心位置の更新 は下記の部分で行なっています。

```jsx
useEffect(() => {
  if (map && options.center) {
    map.setCenter(options.center);
  }
}, [map, options.center]);
```

上記のコードでは、`map` または `options.center` が変更されると、地図の中心位置を更新します。

これにより、地図の中心を動的に変更することができます。

<br>

### マーカーの作成

マーカーを地図上に追加し、マーカーのドラッグやクリックイベントで情報ウィンドウを表示します。

マーカーがドラッグされたときには、新しい位置を設定し、その位置の住所データを取得して情報ウィンドウに表示します。

```jsx
import { useState, useEffect } from "react";
import axios from "axios";
import ReactDOM from "react-dom";
import InfoWindow from "./InfoWindow";

const Marker = (options: google.maps.MarkerOptions & {
  map?: google.maps.Map,
  draggable?: boolean,
  onDragEnd?: (e: google.maps.MapMouseEvent) => void,
}) => {
  
  const [marker, setMarker] = useState<google.maps.Marker>();
  const [infoWindow, setInfoWindow] = useState<google.maps.InfoWindow>();
  
  const [position, setPosition] = useState<{ lat: number; lng: number }>(() => {
	  const pos = options.position;
	    
	  if (pos instanceof google.maps.LatLng) {
	      return {
	        lat: pos.lat(),
	        lng: pos.lng(),
	      };
	    } else if (pos) {
	      return {
	        lat: pos.lat,
	        lng: pos.lng,
	      };
	    }
    return { lat: 0, lng: 0 };
  });

  const fetchAddress = async (lat: number, lng: number) => {
    try {
      const response = await axios.get(`http://localhost:3000/reverse_geocode?lat=${lat}&lng=${lng}`);
      return response.data.address;
    } catch (error) {
      console.error("Error fetching location data:", error);
      alert("住所データの取得に失敗しました");
    }
  };

  useEffect(() => {
    if (!marker && options.map) {
	    const newMarker = new google.maps.Marker({
	      ...options,
	      draggable: options.draggable,
	    });

	    // マーカーのドラッグ終了時、新しい位置情報を取得
	    newMarker.addListener("dragend", async (e: google.maps.MapMouseEvent) => {
	      const newPosition = {
	        lat: e.latLng?.lat() || 0,
	        lng: e.latLng?.lng() || 0,
	      };
	      setPosition(newPosition);
	
	      const address = await fetchAddress(newPosition.lat, newPosition.lng);
	
	      if (infoWindow) {
	        const infoWindowDiv = document.createElement("div");
	        ReactDOM.render(<InfoWindow position={newPosition} address={address} />, infoWindowDiv);
	        infoWindow.setContent(infoWindowDiv);
	        infoWindow.open(options.map, newMarker);
	      } else {
	        const infoWindowDiv = document.createElement("div");
	        ReactDOM.render(<InfoWindow position={newPosition} address={address} />, infoWindowDiv);
	        const newInfoWindow = new google.maps.InfoWindow({
	          content: infoWindowDiv,
	        });
	        newInfoWindow.open(options.map, newMarker);
	        setInfoWindow(newInfoWindow);
	      }
	    });

      // マーカークリック時に現在の位置情報に基づいて住所を取得
	    newMarker.addListener("click", async () => {
	      const address = await fetchAddress(position.lat, position.lng);
	
	      if (infoWindow) {
	        const infoWindowDiv = document.createElement("div");
	        ReactDOM.render(<InfoWindow position={position} address={address} />, infoWindowDiv);
	        infoWindow.setContent(infoWindowDiv);
	        infoWindow.open(options.map, newMarker);
	      } else {
	        const infoWindowDiv = document.createElement("div");
	        ReactDOM.render(<InfoWindow position={position} address={address} />, infoWindowDiv);
	        const newInfoWindow = new google.maps.InfoWindow({
	          content: infoWindowDiv,
	        });
	        newInfoWindow.open(options.map, newMarker);
	        setInfoWindow(newInfoWindow);
	      }
	    });

      setMarker(newMarker);
    }

    // アンマウント時にマーカーをマップから削除（クリーンアップ）
    return () => {
      if (marker) {
        marker.setMap(null);
      }
    };
  }, [marker, options]);

  useEffect(() => {
    if (marker && options.map) {
      marker.setMap(options.map);
      marker.setOptions(options);
    }
  }, [marker, options]);

  return null;
};

export default Marker;
```

ここでの重要なことは大きく2つです。

1. マーカードラッグ時にマーカーが示す座標データから住所データの取得
2. マーカークリック時にマーカーが示す座標データから住所データの取得

<br>
1. マーカードラッグ時にマーカーが示す座標データから住所データの取得 は、下記の箇所で行な

っています。

```jsx
// マーカーのドラッグ終了時、新しい位置情報を取得
newMarker.addListener("dragend", async (e: google.maps.MapMouseEvent) => {
  const newPosition = { lat: e.latLng?.lat() || 0, lng: e.latLng?.lng() || 0 };
  setPosition(newPosition);
  const address = await fetchAddress(newPosition.lat, newPosition.lng);
  ...
});
```

行なっていることとしては、マーカーのドラッグが終了した際に、イベントリスナーで `newPosition`（新しい位置）を取得し、状態を更新しています。

さらに、新しい位置に基づいて `fetchAddress` を使って住所を取得し、その結果を `InfoWindow` （情報ウィンドウ）に表示しています。

<br>
2. マーカークリック時にマーカーが示す座標データから住所データの取得 は、下記の箇所で行な

っています。

```jsx
// マーカークリック時に現在の位置情報に基づいて住所を取得
newMarker.addListener("click", async () => {
  const address = await fetchAddress(position.lat, position.lng);
  ...
});
```

こちらはマーカーがクリックされた際に、現在の位置に基づいて住所を取得し、こちらでもその内容を `InfoWindow` （情報ウィンドウ）に表示しています。

<br>

### 情報ウィンドウの作成

次に、情報ウィンドウの内容を表示するコンポーネントを作成します。

ここでは、緯度、経度、国、郵便番号、市区町村を表示しています。

```jsx
import React from "react";

interface Address {
  country: string;
  postcode: string;
  city: string;
}

interface InfoWindowProps {
  position: google.maps.LatLngLiteral;
  address: Address;
}

const InfoWindow: React.FC<InfoWindowProps> = ({ position, address }) => {
  return (
    <div>
      <p>緯度: {position.lat}</p>
      <p>経度: {position.lng}</p>
      <p>国: {address?.country}</p>
      <p>郵便番号: {address?.postcode}</p>
      <p>市区町村: {address?.city}</p>
    </div>
  );
};

export default InfoWindow;
```

<br>

### 住所情報を取得

最後に、緯度と経度を用いた住所情報を取得するロジックを解説します。

rails 側ではマーカー上の位置情報から取得した緯度と経度を用いて、住所情報を取得しました。

今回、rails で使用できる「**Geocoder**」というgemライブラリを用いてみました。

<br>
Geocoder とは？
Geocoder は、Ruby で使える地理情報に関連する gem（ライブラリ）です。主な役割は、住所から緯度経度を取得する「ジオコーディング」や、逆に緯度経度から住所を取得する「逆ジオコーディング」を行うことです。

https://github.com/alexreisner/geocoder

Geocoder を用いるには、まず Gemfile に gem 'geocoder’ を記述し、bundle install コマンドを実行してください。

Geocoder を用いた緯度と経度情報から住所情報を取得する処理が下記になります。

```ruby
# routes.rb
Rails.application.routes.draw do
  get 'reverse_geocode', to: 'google_maps#reverse_geocode'
end

# google_maps_controller.rb
class GoogleMapsController < ApplicationController

  # 緯度と経度情報から住所情報（リバースジオコーディングの実行）
  def reverse_geocode
    results = Geocoder.search([params[:lat], params[:lng]]).first
    if results
      render json: { address: results.data["address"] }
    else
      render json: { error: '位置が見つかりませんでした' }, status: :not_found
    end
  end
end
```

上記の処理で緯度と経度情報から住所情報を取得し、そのデータをJSON形式でフロント側（今回でいうとTypescript）に返却しています。

以上で Google Mapsでドラッグ可能なピンと情報ウィンドウを表示する方法 の説明は終了です。

<br>

### まとめ

今回、Google Mapsでドラッグ可能なピンと情報ウィンドウを表示する方法 を解説しました。

今回用いた技術的は Typescript と Rails をメインに実装し、Typescript を用いたアプリケーションの作成はほとんど初めてでした。

上記の実装手順により、Google Maps上にドラッグ可能なピンを表示し、ピンをクリックまたはドラッグすると、その位置に基づく住所情報を情報ウィンドウで表示することができます。

上記のコードを参考にして、さらにカスタマイズや機能追加を行ってみてください。

<br>

### 参考

https://developers.google.com/maps/documentation/javascript/get-api-key?hl=ja

https://developers.google.com/maps/documentation/javascript/react-map?hl=ja

https://github.com/alexreisner/geocoder