

```java
// import this to use in Leetcode
import java.util.AbstractMap;
import java.util.Map;

class MulitpleWayToCreatePairInJava {
    public record RecordPair<K, V>(K key, V value) {
        // The compiler automatically generates the constructor, accessor methods (key(), value()),
        // equals(), hashCode(), and toString() methods.
    }

    public static void main(String[] args) {
        Integer key = 1;
        String value = "one";

        // ---------- 3rd party library ----------
        // 1. Using `Pair` class from https://docs.oracle.com/javase/8/javafx/api/javafx/util/Pair.html. (LeetCode includes this)
        Pair<Integer, String> javafxPair = new Pair<>(key, value);
        // Access the components
        int k1 = javafxPair.getKey();
        String v1 = javafxPair.getValue();
        System.out.println("(javafx)Pair : {key="+k1+" --> value="+v1+"}");

        // ---------- No external library ----------

        // 2. Using Object array

        Object[] objArrPair = new Object[] {2, "two"}; // getPair(key, value);

        // Access the components

        int k2 = (Integer) objArrPair[0];

        String v2 = (String) objArrPair[1];

        System.out.println("Object[] : {key="+k2+" --> value="+v2+"}");

        // 3. Using Record : immutable pair
        RecordPair<Integer, String> recordPair = new RecordPair<>(key, value);
        // Access the components
        int k3 = recordPair.key();
        String v3 = recordPair.value();
        System.out.println("record : {key="+k3+" --> value="+v3+"}");

        // 4. Using java.util.AbstractMap (requires explicit import in LeetCode)

        // 4.1: Direct instantiation using the class name
        AbstractMap.SimpleEntry<Integer, String> simplEntry = new AbstractMap.SimpleEntry<>(1, "one");
        Integer keyx = simplEntry.getKey();
        String valuex = simplEntry.getValue();
        System.out.println("AbstractMap.SimpleEntry : {key="+k3+" --> value="+v3+"}");

        // 4.2: Referencing it as a Map.Entry (common in LeetCode)
        Map.Entry<Integer, String> mapEntry = new AbstractMap.SimpleEntry<>(2, "two");
        Integer keyz = mapEntry.getKey();
        String valuez = mapEntry.getValue();
        System.out.println("Map.Entry : {key="+k3+" --> value="+v3+"}");

        return new int[]{0,0};

    }

    private Object[] getPair(Object key, Object value) {
        return new Object[] {key, value};
    }

}
```