# DATA514_Aribnb

#Q1 AirBnB search: Display list of stays in Portland, OR with details: name, #neighbourhood, room type, how many guests it accommodates, property type and #amenities, per night’s cost and is available for the next two days in descending #order of rating. 

#Q2 Are there any neighbourhoods in any of the cities that don’t have any #listings?

#Q3 Availability for booking: For “Entire home/apt” type listings in Salem #provide it’s availability estimate for each month – which chunks of time are #bookable? Display listing’s name, whether it’s Entire home/apt, month, #availability “from – to” date/or just date if minimum nights is 1, and minimum #nights. 


#Q4 Booking trend for Spring v/s Winter: For “Entire home/apt” type listings in #Portland provide it’s availability estimate for each month of Spring and Winter #this year.

#Q5 Booking Trend: For each city, how many reviews are received for December of #each year?
```
db.reviews.aggregate([
  {
    $project: {
        date :1,
  	month: {$month : "$date"},
 	year: { $year: "$date" },
  	city: 1,
  	comments: 1
    }
  },
  {
    $match: {
      $expr: {
        $eq: [{ $month: "$date" }, 12]
      }
    }
  },
  {
    $group: {
      _id: {
        year: "$year",
        city: "$city"
      },
      reviewCount: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      year: "$_id.year",
      city: "$_id.city",
      reviewCount: 1
    }
  },
  {
     $sort: {
       city: 1,
       year: 1

     }
  }
])
```
#Q6 Reminder to Book again: Are there any listings that have received more than #three reviews from the same reviewer within a month? Additionally, are there any #other listings by the same host that can be suggested? If so, please display the #listing's name, URL, description, host's name, reviewer's name, whether it was #previously booked, availability days, minimum nights for booking, and maximum #nights for booking. (Slightly modified from the actual query)

db.reviews.aggregate([
  {
    $group: {
      _id: {
        listing_id: "$listing_id",
        reviewer_id: "$reviewer_id",
        month: { $month: "$date" }
      },
      count: { $sum: 1 }
    }
  },
  {
    $match: {
      count: { $gt: 3 }
    }
  },
  {
    $lookup: {
      from: "listings",
      localField: "_id.listing_id",
      foreignField: "id",
      as: "listing"
    }
  },
  {
    $lookup: {
      from: "reviews",
      localField: "_id.listing_id",
      foreignField: "listing_id",
      as: "previous_reviews"
    }
  },
  {
    $lookup: {
      from: "listings",
      localField: "listing.host_id",
      foreignField: "host_id",
      as: "other_listings"
    }
  },
  {
    $match: {
      "other_listings.id": { $ne: "$_id.listing_id" }
    }
  },
  {
    $project: {
      _id: 0,
      listing_id: "$_id.listing_id",
      reviewer_id: "$_id.reviewer_id",
      month: "$_id.month",
      listing_name: { $arrayElemAt: ["$listing.name", 0] },
      listing_url: { $arrayElemAt: ["$listing.listing_url", 0] },
      listing_description: { $arrayElemAt: ["$listing.description", 0] },
      host_name: { $arrayElemAt: ["$listing.host_name", 0] },
      reviewer_name: { $arrayElemAt: ["$previous_reviews.reviewer_name", 0] },
      previously_booked: { $cond: [{ $gte: ["$count", 4] }, true, false] },
      availability_days: "$listing.availability_365",
      min_nights: "$listing.minimum_nights",
      max_nights: "$listing.maximum_nights",
      other_listings: {
        $map: {
          input: "$other_listings",
          as: "other",
          in: {
            listing_name: "$$other.name",
            listing_url: "$$other.listing_url",
            listing_description: "$$other.description",
            host_name: "$$other.host_name",
            reviewer_name: "$$other.reviewer_name",
            previously_booked: "$$other.previously_booked",
            availability_days: "$$other.availability_365",
            min_nights: "$$other.minimum_nights",
            max_nights: "$$other.maximum_nights"
          }
        }
      }
    }
  }
])

#We’ve identified 2 other questions we would like to add. They are as follows:

#Q7 Hosts with the Most Listings:Display the list of hosts with the most number #of listings.

[
  {
    $group:
      /**
       * grouping documents by host_id field
       * count of listings for each host
       */
      {
        _id: "$host_id",
        count: {
          $sum: 1,
        },
      },
  },
  {
    $sort:
      /**
       * sort in desc order
       */
      {
        count: -1,
      },
  },
  {
    $limit:
      /**
       * top 5 hosts
       */
      5,
  },
]


#Q8 Average Review Scores by Property Type: What is the average review score for #each property type?

[
  {
    $match:
      /**
  filters out documents where 
  review_scores_rating field is "NA" 
  and 
  property_type field is not null.
  */
      {
        review_scores_rating: {
          $ne: "NA",
        },
        property_type: {
          $ne: null,
        },
      },
  },
  {
    $group:
      /**
  groups the documents by the 
  property_type field and 
  calculates the average review scores 
  using the $avg aggregation operator. 
   $toDouble operator is used to 
  convert the review_scores_rating 
  field to a numeric value.
  */
      {
        _id: "$property_type",
        avg_review_scores: {
          $avg: {
            $toDouble: "$review_scores_rating",
          },
        },
      },
  },
]
