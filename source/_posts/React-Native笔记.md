# 

## setState与异步
setState是异步的，在看[React Native Tutorial: Building iOS and Android Apps with JavaScript](https://www.raywenderlich.com/165140/react-native-tutorial-building-ios-android-apps-javascript)的例子的时候，发现

demo 

```
_onSearchTextChanged = (event) => {
    console.log('_onSearchTextChanged current:' + this.state.searchString + ', eventText:' + event.nativeEvent
    .text);
    this.setState( { searchString: event.nativeEvent.text })
    console.log('Current: ' + this.state.searchString+ ', Next:' + event.nativeEvent.text);
}
    
<TextInput
    style={styles.searchInput}
    value={this.state.searchString}
    onChange = {this._onSearchTextChanged}
    placeholder='Search via name or postcode'
/>
```
执行结果如下
![](http://onkcruzxc.bkt.clouddn.com/2017-10-11-15076912164024.jpg)


