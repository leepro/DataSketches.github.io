---
layout: doc_page
---

# Tuple Sketch Java Example

    // simplified file operations and no error handling for clarity

    import java.io.File;
    import java.io.FileInputStream;
    import java.io.FileOutputStream;
    import java.util.Arrays;
    import java.util.Random;

    import com.yahoo.memory.NativeMemory;
    import com.yahoo.sketches.tuple.ArrayOfDoublesSetOperationBuilder;
    import com.yahoo.sketches.tuple.ArrayOfDoublesSketch;
    import com.yahoo.sketches.tuple.ArrayOfDoublesSketchIterator;
    import com.yahoo.sketches.tuple.ArrayOfDoublesSketches;
    import com.yahoo.sketches.tuple.ArrayOfDoublesUnion;
    import com.yahoo.sketches.tuple.ArrayOfDoublesUpdatableSketch;
    import com.yahoo.sketches.tuple.ArrayOfDoublesUpdatableSketchBuilder;
    import com.yahoo.sketches.quantiles.DoublesSketch;

    // this section generates two sketches with some overlap in unique keys
    // and random double values from a normal distribution
    // and serializes them into files in compact (not updatable) form
    {
      Random rand = new Random();

      ArrayOfDoublesUpdatableSketch sketch1 = new ArrayOfDoublesUpdatableSketchBuilder().build();
      for (int key = 0; key < 100000; key++) sketch1.update(key, new double[] {rand.nextGaussian()});
      FileOutputStream out1 = new FileOutputStream(new File("TupleSketch1.bin"));
      out1.write(sketch1.compact().toByteArray());
      out1.close();

      ArrayOfDoublesUpdatableSketch sketch2 = new ArrayOfDoublesUpdatableSketchBuilder().build();
      for (int key = 50000; key < 150000; key++) sketch2.update(key, new double[] {rand.nextGaussian()});
      FileOutputStream out2 = new FileOutputStream(new File("TupleSketch2.bin"));
      out2.write(sketch2.compact().toByteArray());
      out2.close();
    }

    // this section deserializes the sketches, produces union and prints some results
    {
      FileInputStream in1 = new FileInputStream(new File("TupleSketch1.bin"));
      byte[] bytes1 = new byte[in1.available()];
      in1.read(bytes1);
      in1.close();
      ArrayOfDoublesSketch sketch1 = ArrayOfDoublesSketches.wrapSketch(new NativeMemory(bytes1));

      FileInputStream in2 = new FileInputStream(new File("TupleSketch2.bin"));
      byte[] bytes2 = new byte[in2.available()];
      in2.read(bytes2);
      in2.close();
      ArrayOfDoublesSketch sketch2 = ArrayOfDoublesSketches.wrapSketch(new NativeMemory(bytes2));

      ArrayOfDoublesUnion union = new ArrayOfDoublesSetOperationBuilder().buildUnion();
      union.update(sketch1);
      union.update(sketch2);
      ArrayOfDoublesSketch unionResult = union.getResult();

      System.out.println("Union unique count estimate: " + unionResult.getEstimate());
      System.out.println("Union unique count lower bound (95% confidence): " + unionResult.getLowerBound(2));
      System.out.println("Union unique count upper bound (95% confidence): " + unionResult.getUpperBound(2));

      // Let's use Quantiles sketch to analyze the distribution of values
      DoublesSketch quantilesSketch = DoublesSketch.builder().build();
      ArrayOfDoublesSketchIterator it = unionResult.iterator();
      while (it.next()) {
        quantilesSketch.update(it.getValues()[0]);
      }

      System.out.println("Probability Histogram of values: estimated probability mass in 6 bins:\n"
          + "(-inf, -2), [-2, -1), [-1, 0), [0, 1), [1, 2), [2, +inf)");
      System.out.println(Arrays.toString(quantilesSketch.getPMF(new double[] {-2, -1, 0, 1, 2})));
    }

    Output:
    Union unique count estimate: 149586.73149344584
    Union unique count lower bound (95% confidence): 145028.6046846571
    Union unique count upper bound (95% confidence): 154287.5017892762
    Probability Histogram of values: estimated probability mass in 6 bins:
    (-inf, -2), [-2, -1), [-1, 0), [0, 1), [1, 2), [2, +inf)
    [0.0390625, 0.1484375, 0.3125, 0.3046875, 0.1484375, 0.046875]
