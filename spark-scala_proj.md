Creating an end-to-end Apache Spark project with Scala that reads data from S3, performs cleaning, auditing, data quality checks, joins, saves the results to Redshift, and sends alerts for performance monitoring involves several steps. Below is a simplified example to guide you through the process.

### 1. Project Setup

First, set up your project using Maven. Here’s a basic `pom.xml` file:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>spark-scala-project</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <scala.version>2.12.10</scala.version>
        <spark.version>3.1.2</spark.version>
    </properties>

    <dependencies>
        <!-- Spark Core -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.12</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- Spark SQL -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.12</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- AWS SDK for S3 -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-aws</artifactId>
            <version>3.2.0</version>
        </dependency>

        <!-- Redshift JDBC -->
        <dependency>
            <groupId>com.amazon.redshift</groupId>
            <artifactId>redshift-jdbc42</artifactId>
            <version>1.2.1.1001</version>
        </dependency>

        <!-- For sending alerts (e.g., via email) -->
        <dependency>
            <groupId>javax.mail</groupId>
            <artifactId>mail</artifactId>
            <version>1.4.7</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>4.5.6</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2. Scala Code

Here’s a simplified Scala code that performs the required operations:

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import java.util.Properties
import javax.mail._
import javax.mail.internet._

object SparkS3Redshift {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("Spark S3 to Redshift")
      .config("spark.hadoop.fs.s3a.access.key", "YOUR_ACCESS_KEY")
      .config("spark.hadoop.fs.s3a.secret.key", "YOUR_SECRET_KEY")
      .config("spark.hadoop.fs.s3a.endpoint", "s3.amazonaws.com")
      .getOrCreate()

    // Read data from S3
    val df = spark.read.option("header", "true").csv("s3a://your-bucket-name/your-data.csv")

    // Data Cleaning
    val cleanedDF = df.na.drop().filter(col("column_name").isNotNull)

    // Data Auditing
    val rowCount = cleanedDF.count()
    println(s"Total rows after cleaning: $rowCount")

    // Data Quality Check
    val invalidRows = cleanedDF.filter(col("column_name").rlike("your_regex_pattern"))
    if (invalidRows.count() > 0) {
      println("Data quality check failed: Invalid rows found")
      sendAlert("Data quality check failed: Invalid rows found")
    }

    // Join with another dataset
    val anotherDF = spark.read.option("header", "true").csv("s3a://your-bucket-name/another-data.csv")
    val joinedDF = cleanedDF.join(anotherDF, Seq("join_key"))

    // Save to Redshift
    val redshiftURL = "jdbc:redshift://your-redshift-cluster:5439/your-database"
    val properties = new Properties()
    properties.setProperty("user", "your-username")
    properties.setProperty("password", "your-password")
    properties.setProperty("driver", "com.amazon.redshift.jdbc42.Driver")

    joinedDF.write
      .mode("overwrite")
      .jdbc(redshiftURL, "your_redshift_table", properties)

    // Performance Monitoring Alert
    if (rowCount < 1000) {
      sendAlert(s"Low row count after processing: $rowCount")
    }

    spark.stop()
  }

  def sendAlert(message: String): Unit = {
    val from = "your-email@example.com"
    val to = "recipient-email@example.com"
    val host = "smtp.example.com"

    val properties = System.getProperties
    properties.setProperty("mail.smtp.host", host)

    val session = Session.getDefaultInstance(properties)
    val mimeMessage = new MimeMessage(session)
    mimeMessage.setFrom(new InternetAddress(from))
    mimeMessage.addRecipient(Message.RecipientType.TO, new InternetAddress(to))
    mimeMessage.setSubject("Spark Job Alert")
    mimeMessage.setText(message)

    Transport.send(mimeMessage)
  }
}
```

### 3. Explanation

- **Reading from S3**: The code reads a CSV file from an S3 bucket using the `s3a` protocol.
- **Data Cleaning**: Drops null values and filters out rows with invalid data.
- **Data Auditing**: Counts the number of rows after cleaning.
- **Data Quality Check**: Checks for invalid rows using a regex pattern and sends an alert if any are found.
- **Joining Data**: Joins the cleaned data with another dataset from S3.
- **Saving to Redshift**: Writes the joined data to a Redshift table using JDBC.
- **Performance Monitoring Alert**: Sends an email alert if the row count is below a certain threshold.

### 4. Running the Project

To run the project, use the following command:

```bash
mvn clean package
spark-submit --class com.example.SparkS3Redshift --master yarn --deploy-mode cluster target/spark-scala-project-1.0-SNAPSHOT.jar
```

### 5. Notes

- Replace placeholders like `YOUR_ACCESS_KEY`, `YOUR_SECRET_KEY`, `your-bucket-name`, `your-redshift-cluster`, etc., with actual values.
- Ensure that the necessary AWS credentials and Redshift permissions are configured.
- The email alert functionality uses JavaMail. You may need to configure your SMTP server settings.

This is a basic example to get you started. Depending on your specific requirements, you may need to add more sophisticated error handling, logging, and performance optimizations.